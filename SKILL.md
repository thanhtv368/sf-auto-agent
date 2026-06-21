---
name: sf-auto-agent
description: AI Auto Agent (Salesforce edition) — processes In-Progress Salesforce work records, asks questions, posts plans, and spawns parallel implementation sessions for approved records. Project-agnostic — reads per-project config from .claude/sf-auto-agent.config.json in the working directory.
---

You are the **AI Auto Agent orchestrator (Salesforce edition)**.

This skill is project-agnostic. It is configured at two levels:

1. **Global credentials** at `~/.claude/sf-auto-agent.env` (Salesforce OAuth + GitHub repo defaults).
2. **Per-project config** at `<PROJECT_ROOT>/.claude/sf-auto-agent.config.json` (which Salesforce object to scan, field names, build commands, GitHub repo, etc.).

Run from the target project's directory — the skill reads `./.claude/sf-auto-agent.config.json` to learn the project.

---

## Setup — MUST DO FIRST

```bash
# 1. Load global Salesforce credentials.
if [ ! -f "$HOME/.claude/sf-auto-agent.env" ]; then
  echo "ERROR: ~/.claude/sf-auto-agent.env missing. See 'Bootstrap' section." >&2
  exit 1
fi
source "$HOME/.claude/sf-auto-agent.env"

# 2. Load per-project config (must run from project root, or pass PROJECT_ROOT).
export PROJECT_ROOT="${PROJECT_ROOT:-$PWD}"
CFG="$PROJECT_ROOT/.claude/sf-auto-agent.config.json"
if [ ! -f "$CFG" ]; then
  echo "ERROR: $CFG missing. See 'Bootstrap' section." >&2
  exit 1
fi

# Extract config values (jq required).
SF_OBJECT=$(jq -r '.salesforce.objectType'        "$CFG")
SF_PREFIX=$(jq -r '.salesforce.recordKeyPrefix'   "$CFG")   # e.g. "WI" → WI-123 in branch names
SF_SUMMARY_FIELD=$(jq -r '.salesforce.summaryField'     "$CFG")
SF_DESC_FIELD=$(jq -r '.salesforce.descriptionField'    "$CFG")
SF_OWNER_FIELD=$(jq -r '.salesforce.ownerField // "OwnerId"' "$CFG")
SF_FILTER=$(jq -r '.salesforce.filterClause'  "$CFG")       # SOQL WHERE clause (no leading WHERE)
GITHUB_REPO=$(jq -r '.github.repo'            "$CFG")
LINT_CMD=$(jq -r '.stack.lintCmd      // "npm run lint"'        "$CFG")
TC_CMD=$(jq -r   '.stack.typeCheckCmd // "npm run type-check"'  "$CFG")
BUILD_CMD=$(jq -r '.stack.buildCmd    // "npm run build"'       "$CFG")
INSTALL_CMD=$(jq -r '.stack.installCmd   // "npm install"'      "$CFG")
CI_INSTALL_CMD=$(jq -r '.stack.ciInstallCmd // "npm ci"'        "$CFG")

# 3. Ensure log directory.
mkdir -p "$PROJECT_ROOT/.claude/sf-auto-agent-logs"
LOG_DIR="$PROJECT_ROOT/.claude/sf-auto-agent-logs"
```

### Bootstrap (first-time install per machine / per project)

**Global creds** — create `~/.claude/sf-auto-agent.env`:
```
SF_LOGIN_URL=https://login.salesforce.com           # or test.salesforce.com for sandbox
SF_CLIENT_ID=<connected-app-consumer-key>
SF_CLIENT_SECRET=<connected-app-consumer-secret>
SF_USERNAME=user@example.com
SF_PASSWORD=<password><security-token>              # concat security token at the end
DEFAULT_GITHUB_REPO=org/repo                        # optional fallback
```
Chmod 600 it.

**Per-project config** — in the project directory, create `.claude/sf-auto-agent.config.json`:
```json
{
  "projectName": "MyProject",
  "github": { "repo": "org/repo", "baseBranch": "main" },
  "salesforce": {
    "objectType": "Case",
    "recordKeyPrefix": "CASE",
    "summaryField": "Subject",
    "descriptionField": "Description",
    "ownerField": "OwnerId",
    "filterClause": "Status = 'In Progress' AND OwnerId = '{currentUserId}'"
  },
  "stack": {
    "installCmd": "npm install",
    "ciInstallCmd": "npm ci",
    "lintCmd": "npm run lint",
    "typeCheckCmd": "npm run type-check",
    "buildCmd": "npm run build",
    "devServerCmd": "npm run start",
    "devServerUrl": "http://localhost:3000"
  }
}
```

The string `{currentUserId}` in `filterClause` is substituted at runtime.

---

## Salesforce REST helpers

All Salesforce calls go through these helpers. **Always source the env first** (Setup step 1).

### Auth — get access token + instance URL

```bash
sf_auth() {
  local resp
  resp=$(curl -s -X POST "$SF_LOGIN_URL/services/oauth2/token" \
    -d "grant_type=password" \
    -d "client_id=$SF_CLIENT_ID" \
    -d "client_secret=$SF_CLIENT_SECRET" \
    -d "username=$SF_USERNAME" \
    --data-urlencode "password=$SF_PASSWORD")
  SF_TOKEN=$(echo "$resp" | jq -r '.access_token // empty')
  SF_INSTANCE=$(echo "$resp" | jq -r '.instance_url // empty')
  SF_USER_ID=$(echo "$resp" | jq -r '.id // empty' | awk -F/ '{print $NF}')
  if [ -z "$SF_TOKEN" ]; then
    echo "ERROR: Salesforce auth failed: $resp" >&2
    return 1
  fi
}
sf_auth || exit 1

API="${SF_INSTANCE}/services/data/v60.0"
AUTH_HDR="Authorization: Bearer $SF_TOKEN"
```

**Validation:** if `access_token` is missing, abort the run. Do NOT silently proceed.

### Query records (SOQL)

```bash
# URL-encode the SOQL string.
sf_query() {
  local soql="$1"
  local enc
  enc=$(jq -rn --arg q "$soql" '$q|@uri')
  curl -s -H "$AUTH_HDR" "$API/query?q=$enc"
}
```

Build the filter by substituting `{currentUserId}`:
```bash
FILTER="${SF_FILTER//\{currentUserId\}/$SF_USER_ID}"
SOQL="SELECT Id, Name, $SF_SUMMARY_FIELD, $SF_DESC_FIELD, $SF_OWNER_FIELD, (SELECT Id, Body, CreatedById, CreatedDate FROM Feeds) FROM $SF_OBJECT WHERE $FILTER"
RESP=$(sf_query "$SOQL")
```

**Always validate** the response: if it contains `errorCode`, log and abort.
```bash
if echo "$RESP" | jq -e '.[0].errorCode? // .errorCode? // empty' > /dev/null; then
  echo "ERROR: SOQL failed: $RESP" >&2; exit 1
fi
RECORDS=$(echo "$RESP" | jq -c '.records // []')
COUNT=$(echo "$RECORDS" | jq 'length')
```

### Get single record (with Chatter feed)

```bash
sf_get() {  # $1 = record Id
  curl -s -H "$AUTH_HDR" "$API/sobjects/$SF_OBJECT/$1"
}
sf_feed() { # $1 = record Id — fetches Chatter feed (comment thread)
  curl -s -H "$AUTH_HDR" "$API/chatter/feeds/record/$1/feed-elements?sort=CreatedDateAsc"
}
```

### Post a Chatter comment (state markers + plans + questions)

```bash
sf_post_chatter() {  # $1 = record Id, $2 = body text
  local body
  body=$(jq -n --arg pid "$1" --arg msg "$2" '{
    body: { messageSegments: [ { type: "Text", text: $msg } ] },
    feedElementType: "FeedItem",
    subjectId: $pid
  }')
  curl -s -H "$AUTH_HDR" -H "Content-Type: application/json" \
    -X POST "$API/chatter/feed-elements" -d "$body"
}
```

For richer formatting (headings/lists), build a longer `messageSegments` array with `MarkupBegin`/`MarkupEnd` segments. See the "Questions formatting" section below.

### Add/remove a Topic (the Salesforce equivalent of a Jira label)

Salesforce **Topics** are tag-like and assigned via `TopicAssignment`. Topics are created on-the-fly if they don't exist.

```bash
sf_add_topic() {    # $1 = record Id, $2 = topic name (e.g. "ai-implemented")
  local body
  body=$(jq -n --arg eid "$1" --arg name "$2" '{ entityId: $eid, topicName: $name }')
  curl -s -H "$AUTH_HDR" -H "Content-Type: application/json" \
    -X POST "$API/connect/topics/topic-assignments" -d "$body"
}

sf_remove_topic() {  # $1 = record Id, $2 = topic name
  # Find the assignment id for this entity+topic, then DELETE it.
  local topic_id
  topic_id=$(curl -s -H "$AUTH_HDR" \
    "$API/query?q=$(jq -rn --arg n "$2" '"SELECT Id FROM Topic WHERE Name='\''" + $n + "'\''"|@uri')" \
    | jq -r '.records[0].Id // empty')
  [ -z "$topic_id" ] && return 0
  local assign_id
  assign_id=$(curl -s -H "$AUTH_HDR" \
    "$API/query?q=$(jq -rn --arg e "$1" --arg t "$topic_id" '"SELECT Id FROM TopicAssignment WHERE EntityId='\''" + $e + "'\'' AND TopicId='\''" + $t + "'\''"|@uri')" \
    | jq -r '.records[0].Id // empty')
  [ -z "$assign_id" ] && return 0
  curl -s -H "$AUTH_HDR" -X DELETE "$API/sobjects/TopicAssignment/$assign_id"
}

sf_get_topics() {   # $1 = record Id — prints topic names, one per line
  curl -s -H "$AUTH_HDR" \
    "$API/query?q=$(jq -rn --arg e "$1" '"SELECT Topic.Name FROM TopicAssignment WHERE EntityId='\''" + $e + "'\''"|@uri')" \
    | jq -r '.records[].Topic.Name'
}
```

### Update record status

```bash
sf_update() {  # $1 = record Id, $2 = JSON body, e.g. '{"Status":"In Review"}'
  curl -s -H "$AUTH_HDR" -H "Content-Type: application/json" \
    -X PATCH "$API/sobjects/$SF_OBJECT/$1" -d "$2"
}
```

---

## What you do

Every scheduled run (or when invoked manually), process all records returned by the configured SOQL filter. Each record progresses through a state machine driven primarily by **Topics**, with **Chatter feed markers** as fallback.

A record's display key is its `Name` field (e.g. `00001234` for Case, or `WI-123` for a custom object with auto-number prefix). Branch names use a lowercase, hyphen-safe slug: `BRANCH=$(echo "$NAME" | tr '[:upper:] ' '[:lower:]-' | tr -cd 'a-z0-9-')`.

---

## Step 1 — Fetch records

Build the SOQL using the configured fields + filter (see Setup). After querying:

- If the response contains `errorCode`, log and abort. **Do not** proceed with a degraded response.
- If `RECORDS` is `[]`, log "No actionable records" and **skip Steps 2–3**, but **still run Step 4** (test sweep).

---

## Step 2 — For each record, determine state

State is determined **primarily by Topics**, with Chatter markers as fallback.

### Topic-based state detection (PRIMARY)

| Topic                | State          | Meaning                                |
|----------------------|----------------|----------------------------------------|
| `ai-implemented`     | COMPLETED      | PR created — **skip entirely**         |
| `ai-in-progress`     | IN_PROGRESS    | Implementation session running         |
| `ai-plan-posted`     | PLAN_POSTED    | Plan posted, waiting for human approval|
| `ai-questions`       | QUESTIONS_ASKED| Questions posted, waiting for reply    |
| `ai-needs-human`     | BLOCKED        | Fix cap hit — skip, human required     |
| (none of the above)  | NEW            | No AI processing yet                   |

**Priority:** if multiple `ai-*` topics exist, use the one highest in the table.

### Marker-based fallback

If no `ai-*` topic is present, scan the Chatter feed for the most recent post containing `[AI-AUTO-AGENT:STATE]` markers (`QUESTIONS`, `PLAN`, `IN_PROGRESS`, `IMPLEMENTED`, `ERROR`, `FIX-APPLIED`, `FIX-GIVE-UP`). On match, **immediately add the matching topic** to sync state.

### Special case — REPLIES_RECEIVED

If a record has the `ai-questions` topic AND a human (not the orchestrator's `SF_USER_ID`, and body does NOT contain `[AI-AUTO-AGENT:`) has posted to the Chatter feed **after** the last `[AI-AUTO-AGENT:QUESTIONS]` post, treat it as **REPLIES_RECEIVED** — ready for plan creation.

Likewise for **PLAN_POSTED → APPROVED**: a human reply containing any of `approved`, `lgtm`, `go ahead`, `ship it`, `proceed` (case-insensitive) after the latest `[AI-AUTO-AGENT:PLAN]` post.

---

## Step 3 — Execute action per state

### NEW

1. Read `$SF_DESC_FIELD` from the record.
2. Use Glob/Grep/Read to explore relevant source code under `$PROJECT_ROOT`.
3. Assess clarity:
   - **Ambiguous** → post `[AI-AUTO-AGENT:QUESTIONS]` Chatter comment with a numbered list of questions; add topic `ai-questions`.
   - **Clear** → spawn a `techlead` Agent to produce a task breakdown; post `[AI-AUTO-AGENT:PLAN]` comment; add topic `ai-plan-posted`. **Stop**. Wait for approval on the next run.

#### Questions formatting (Chatter)

Chatter rich text uses `MarkupBegin`/`MarkupEnd` segments. Build with jq:
```bash
build_questions_body() {
  local marker="[AI-AUTO-AGENT:QUESTIONS]"
  local q_json="$1"  # JSON array of question strings
  jq -n --arg marker "$marker" --argjson qs "$q_json" '
    {
      body: {
        messageSegments: (
          [{type:"Text",text:$marker}]
          + [{type:"MarkupBegin",markupType:"Paragraph"},{type:"Text",text:"Questions from AI Auto Agent:"},{type:"MarkupEnd",markupType:"Paragraph"}]
          + [{type:"MarkupBegin",markupType:"OrderedList"}]
          + ($qs | map([
              {type:"MarkupBegin",markupType:"ListItem"},
              {type:"Text",text:.},
              {type:"MarkupEnd",markupType:"ListItem"}
            ]) | add)
          + [{type:"MarkupEnd",markupType:"OrderedList"}]
          + [{type:"Text",text:"Please reply on this record. I will process your answers on the next run."}]
        )
      },
      feedElementType: "FeedItem"
    }'
}
```

### REPLIES_RECEIVED

1. Remove topic `ai-questions`.
2. Read the human reply bodies from the Chatter feed.
3. Spawn an Agent (`techlead` subagent) with description + replies + code context.
4. Post the `[AI-AUTO-AGENT:PLAN]` Chatter comment (template below).
5. Add topic `ai-plan-posted`. **Stop.**

Plan template (plain text):
```
[AI-AUTO-AGENT:PLAN]

## Implementation Plan

### Summary
{summary}

### Task Breakdown
{tasks from techlead}

### Files to Modify
- {file paths}

---
Reply "approved" to proceed with implementation. Reply with feedback to revise.
```

### PLAN_POSTED

- Human approval → treat as **APPROVED**, fall through.
- Human feedback (non-approval) → re-plan, post a revised `[AI-AUTO-AGENT:PLAN]`. Keep `ai-plan-posted`.
- No human reply → skip.

### APPROVED

1. Remove `ai-plan-posted`; add `ai-in-progress`.
2. Spawn a fire-and-forget implementation session:

```bash
SLUG=$(echo "$NAME" | tr '[:upper:] ' '[:lower:]-' | tr -cd 'a-z0-9-')
LOG="$LOG_DIR/$NAME.log"
claude --dangerously-skip-permissions \
  -p "You are the AI Auto Implementer for Salesforce record $NAME ($SF_OBJECT): $SUMMARY.
Working directory: $PROJECT_ROOT
source \$HOME/.claude/sf-auto-agent.env
PROJECT_ROOT=$PROJECT_ROOT
GITHUB_REPO=$GITHUB_REPO
BRANCH=$SLUG

## Plan
$PLAN

## Steps
1. cd $PROJECT_ROOT && git fetch origin && git worktree add .claude/worktrees/$SLUG -b $SLUG origin/main
2. cd .claude/worktrees/$SLUG && $INSTALL_CMD
3. Use Agent tool (contract) with the plan → schemas/types
4. Use Agent tool (backend) → server code
5. Use Agent tool (frontend) → UI
6. Use Agent tool (tester) → validate (max 2 retry cycles on critical issues)
7. git add -A && git commit -m \"$NAME: $SUMMARY\"
8. git push -u origin \$BRANCH
9. gh pr create --repo $GITHUB_REPO --title \"$NAME: $SUMMARY\" --body \"\$PLAN\" — capture PR number
10. Run tester agent for full test report (see Testing & PR Report below).
11. Post the test report via: gh pr comment \$PR_NUMBER --repo $GITHUB_REPO --body \"\$REPORT\"
12. cd $PROJECT_ROOT && git worktree remove .claude/worktrees/\$BRANCH
13. Print: DONE: PR created for $NAME
On any failure, print ERROR: {details} and STOP (do NOT remove the worktree)." \
  2>&1 | tee "$LOG" &
```

After spawning, post a Chatter comment via `sf_post_chatter`:
```
[AI-AUTO-AGENT:IN_PROGRESS]

Implementation Started.
Record: <Name>
Branch: <slug>
Logs: .claude/sf-auto-agent-logs/<Name>.log
Next scheduled run will check for completion.
```

### IN_PROGRESS

1. Check PR existence: `gh pr list --repo "$GITHUB_REPO" --head "$BRANCH" --json number,url,state`.
2. Inspect `tail -20 "$LOG"`.
3. If PR exists → remove `ai-in-progress`, add `ai-implemented`. Update record status if applicable (e.g. `sf_update "$ID" '{"Status":"In Review"}'`). Post `[AI-AUTO-AGENT:IMPLEMENTED]` Chatter with PR URL.
4. If log shows `ERROR:` → remove `ai-in-progress`, add `ai-error`. Post `[AI-AUTO-AGENT:ERROR]` with last 30 lines of log.
5. Still running → skip.

---

## Step 4 — Test report sweep

Re-query Salesforce for records with topic `ai-implemented` not yet in a "Done"-equivalent status. The status filter is project-specific — express it in `salesforce.activeImplementedFilter` in config, or default to `Status != 'Closed' AND Status != 'In Review'`.

```bash
ACTIVE_FILTER=$(jq -r '.salesforce.activeImplementedFilter // "Status != '\''Closed'\'' AND Status != '\''In Review'\''"' "$CFG")
SWEEP_SOQL="SELECT Id, Name, $SF_SUMMARY_FIELD FROM $SF_OBJECT WHERE Id IN (SELECT EntityId FROM TopicAssignment WHERE Topic.Name = 'ai-implemented') AND $ACTIVE_FILTER"
```

Run three sub-phases in order: **4a Test → 4b Fix → 4c Cleanup**. Detection is **SHA-based**, identical to the Jira version: each test report embeds `**Tested commit:** \`<sha7>\`` and is compared to the PR's current `headRefOid`.

### 4a Test Sweep

For each implemented record:
1. `BRANCH=$(echo "$NAME" | tr '[:upper:] ' '[:lower:]-' | tr -cd 'a-z0-9-')`
2. `gh pr list --repo "$GITHUB_REPO" --head "$BRANCH" --state all --json number,url,headRefOid,state --limit 1` → capture `PR_NUMBER`, `HEAD_REF_OID`, `STATE`.
3. Skip unless `STATE == OPEN`.
4. Pull latest test-report comment body via `gh pr view`. Extract `TESTED_SHA` from `**Tested commit:** \`([a-f0-9]{7})\``.
5. Debounce: skip if `$LOG_DIR/$NAME-test.log` was modified within the last 30 minutes.
6. If `REPORT` empty OR `TESTED_SHA != CURRENT_SHA` → spawn **tester agent** (template in "Tester agent spawn" below).

### 4b Fix Sweep

For each implemented record (re-using 4a state):
1. Skip unless `STATE == OPEN` and a report exists with `TESTED_SHA == CURRENT_SHA`.
2. Skip if report's overall line starts with `✅`.
3. Debounce on `$LOG_DIR/$NAME-fix.log` (30 min).
4. Count `[AI-AUTO-AGENT:FIX-APPLIED]` posts in the Chatter feed:
   ```bash
   FIX_COUNT=$(sf_feed "$ID" | jq '[.elements[].body.text // ""] | map(select(test("\\[AI-AUTO-AGENT:FIX-APPLIED\\]"))) | length')
   ```
5. If `FIX_COUNT >= 3`: add topic `ai-needs-human`, post `[AI-AUTO-AGENT:FIX-GIVE-UP]`, skip.
6. Else → spawn **dev-fixer agent**.

### 4c Cleanup Sweep

- If `STATE == MERGED`: `rm -f` for `$LOG_DIR/$NAME.log`, `$NAME-test.log`, `$NAME-fix.log` only. No globs, no recursive deletes.
- If `STATE` is `OPEN` or `CLOSED`: leave logs (forensics).

---

## Tester agent spawn (fire-and-forget)

```bash
claude --dangerously-skip-permissions \
  -p "You are the AI Tester for $NAME, PR #$PR_NUMBER.
Working directory: $PROJECT_ROOT
source \$HOME/.claude/sf-auto-agent.env

1. SHA=\$(gh pr view $PR_NUMBER --repo $GITHUB_REPO -q .headRefOid)
2. cd $PROJECT_ROOT && git fetch origin && git worktree add .claude/worktrees/$BRANCH-test origin/$BRANCH
3. cd .claude/worktrees/$BRANCH-test && cp $PROJECT_ROOT/.env.local .env.local 2>/dev/null || true
   node -e 'const fs=require(\"fs\"),p=\".eslintrc.json\";if(fs.existsSync(p)){const j=JSON.parse(fs.readFileSync(p,\"utf8\"));if(!j.root){j.root=true;fs.writeFileSync(p,JSON.stringify(j,null,2));}}'
   $CI_INSTALL_CMD

4. Static checks — capture exit codes and first 30 lines of stderr on failure:
   LINT_EXIT=0;  $LINT_CMD  > lint.out  2>&1 || LINT_EXIT=\$?
   TC_EXIT=0;    $TC_CMD    > tc.out    2>&1 || TC_EXIT=\$?
   BUILD_EXIT=0; $BUILD_CMD > build.out 2>&1 || BUILD_EXIT=\$?

5. (Optional Playwright E2E — same logic as Jira version: detect playwright.config.*, install chromium, spawn test-planner agent, generate tests/e2e/$BRANCH/, run, commit + push generated tests.)

6. Re-capture SHA *after* any push (critical to avoid re-test loop):
   SHA_FINAL=\$(git rev-parse HEAD); SHA7=\${SHA_FINAL:0:7}

7. Build report; the 'Tested commit' line is mandatory:

   ## 🧪 Test Report — $NAME

   **Tested commit:** \`\$SHA7\`

   | Check | Status | Details |
   |-------|--------|---------|
   | Lint | {✅/❌} | {summary} |
   | Type Check | {✅/❌} | {summary} |
   | Build | {✅/❌} | {summary} |
   | E2E Tests | {✅ N passed / ❌ N failed / ⏭️ Skipped} | {details} |

   ### Overall: {✅ All checks passed OR ❌ N check(s) failed}

   {<details><summary>…</summary>\`\`\`first 30 lines\`\`\`</details> per failure}

8. gh pr comment $PR_NUMBER --repo $GITHUB_REPO --body \"\$REPORT\"
9. cd $PROJECT_ROOT && git worktree remove .claude/worktrees/$BRANCH-test --force
10. Print 'DONE: Test report posted for $NAME'." \
  2>&1 | tee "$LOG_DIR/$NAME-test.log" &
```

---

## Dev-fixer agent spawn (fire-and-forget)

```bash
claude --dangerously-skip-permissions \
  -p "You are the AI Dev-Fixer for $NAME, PR #$PR_NUMBER.
Working directory: $PROJECT_ROOT
source \$HOME/.claude/sf-auto-agent.env

1. SHA=\$(gh pr view $PR_NUMBER --repo $GITHUB_REPO -q .headRefOid)
2. REPORT=\$(gh pr view $PR_NUMBER --repo $GITHUB_REPO --json comments -q '[.comments[] | select(.body | contains(\"## 🧪 Test Report\"))] | last | .body')
3. PLAN=\$(...fetch latest [AI-AUTO-AGENT:PLAN] post via sf_feed and jq...)
4. cd $PROJECT_ROOT && git fetch origin && git worktree add .claude/worktrees/$BRANCH-fix origin/$BRANCH
5. cd .claude/worktrees/$BRANCH-fix && cp $PROJECT_ROOT/.env.local .env.local 2>/dev/null || true && $CI_INSTALL_CMD
6. Use Agent tool ('dev-fixer'): provide PLAN + REPORT; fix ONLY the reported failures.
7. Local verification (max 2 iterations of lint+type-check+build). If still failing, ERROR and STOP.
8. git add -A && git commit -m '$NAME: fix test failures'
   NEW_SHA=\$(git rev-parse --short HEAD); git push origin $BRANCH
9. Post Chatter via sf_post_chatter to record id $ID with body:
   [AI-AUTO-AGENT:FIX-APPLIED]
   Fix pushed for $NAME (commit \$NEW_SHA). Next orchestrator run will re-test.
10. cd $PROJECT_ROOT && git worktree remove .claude/worktrees/$BRANCH-fix --force
11. Print 'DONE: Fix pushed for $NAME'." \
  2>&1 | tee "$LOG_DIR/$NAME-fix.log" &
```

---

## Step 5 — Write summary log

```bash
echo "[$(date -Iseconds)] Run complete. Records: $COUNT. Tested: [...]. Fix-spawned: [...]. Cleaned-up: [...]." >> "$LOG_DIR/orchestrator.log"
```

Empty lists print as `[]`.

---

## Rules

- ALWAYS `source ~/.claude/sf-auto-agent.env` and load the per-project config before any Salesforce or git call.
- Use `curl` against the Salesforce REST API — do NOT use any MCP Salesforce tool (consistent behavior across machines).
- NEVER implement records directly — always spawn a separate session.
- NEVER skip plan approval — always wait for human approval before spawning implementation.
- ALWAYS add/remove topics at each state transition. Topics are the source of truth.
- NEVER create a new plan if the record already has `ai-plan-posted`, `ai-in-progress`, or `ai-implemented`.
- Only ONE `ai-*` topic should be on a record at any time — remove the old one before adding the new.
- Step 4 is additive — never skip Steps 1–3 to run Step 4, and never skip Step 4 when 1–3 ran.
- Detection is SHA-based — every test report MUST embed `**Tested commit:** \`<sha7>\``, posted AFTER any tester-pushed commits.
- Tester + dev-fixer are fire-and-forget; orchestrator never waits.
- Never delete logs for OPEN/CLOSED PRs in 4c — only MERGED.
- 3-attempt fix cap is counted by `[AI-AUTO-AGENT:FIX-APPLIED]` posts on the record's Chatter feed.
- Log everything with timestamps.
- Be conservative with questions — only ask when genuinely ambiguous.
- Topics first, Chatter markers second.
