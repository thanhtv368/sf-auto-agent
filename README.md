# sf-auto-agent

A project-agnostic Claude Code skill that turns Salesforce records into PRs.

It scans your "In Progress" work records (Cases, custom Work Items, anything queryable), asks clarifying questions, posts an implementation plan, waits for human approval, then spawns background Claude sessions that implement the work, open a PR, run tests, post a test report, and self-heal failing checks — all while you sleep.

It is a port of an internal Jira-driven agent. The state machine, sweeps, and SHA-based test loop are identical; only the work-tracking backend (Jira → Salesforce) and per-project config layer are different.

---

## What you get

After setup, every scheduled run will:

1. Query Salesforce for records assigned to you in "In Progress".
2. Classify each record by **Topic** (`ai-questions`, `ai-plan-posted`, `ai-in-progress`, `ai-implemented`, `ai-needs-human`) or by Chatter feed markers.
3. Take the next correct action:
   - **NEW** → either ask questions (if ambiguous) or post a plan.
   - **REPLIES_RECEIVED** → re-plan with the human's answers.
   - **APPROVED** → spawn a background implementation session that produces a PR.
   - **IN_PROGRESS** → check if the PR is up; if yes, mark `ai-implemented`.
   - **IMPLEMENTED w/ open PR** → run a tester, post a test report; if failing, run a dev-fixer (up to 3 attempts).
   - **MERGED** → clean up local logs.
4. Log a one-line summary to `.claude/sf-auto-agent-logs/orchestrator.log`.

Topics are the source of truth — every state transition adds/removes one.

---

## Prerequisites

| Tool | Why |
|------|-----|
| Claude Code CLI | Hosts the skill and spawns sub-sessions |
| `gh` (GitHub CLI), authenticated | Opens PRs, posts test-report comments |
| `git` ≥ 2.5 | Worktrees |
| `jq` | All Salesforce JSON parsing |
| `curl` | Salesforce REST calls |
| A Salesforce org you can develop in | To create the Connected App |
| A GitHub repo to push PRs to | Per-project, configurable |

Optional: `playwright` in the target project — if `playwright.config.{ts,js}` exists, the tester will generate + run E2E specs.

---

## Step 1 — Create a Salesforce Connected App

You need OAuth credentials. In your Salesforce org:

1. **Setup** → **App Manager** → **New Connected App**.
2. Basic info: any name (e.g. `sf-auto-agent`), your email.
3. **Enable OAuth Settings** ✓
   - **Callback URL**: `http://localhost/callback` (unused for the password flow, but required).
   - **Selected OAuth Scopes**: at minimum `Manage user data via APIs (api)` and `Perform requests at any time (refresh_token, offline_access)`.
   - ✓ **Enable for Device Flow** (optional, lets you swap to a more secure flow later).
4. Save. Wait ~10 minutes for it to propagate.
5. Open the app → **Manage** → **Edit Policies**:
   - **Permitted Users**: `All users may self-authorize`.
   - **IP Relaxation**: `Relax IP restrictions` (or whitelist your runner's IPs).
6. Note the **Consumer Key** (client id) and **Consumer Secret**.

Then get your **Security Token**:

- Top-right profile → **Settings** → **My Personal Information** → **Reset My Security Token**.
- Salesforce emails it. Append it to your password with **no separator** in the env file below.

If your org enforces MFA on the API user, the username-password flow is blocked. Use a dedicated integration user without MFA, or switch to the JWT bearer flow (the skill's `sf_auth` helper is the only thing that needs to change).

---

## Step 2 — Install the skill and the agents

The repo ships **two** things Claude needs:

1. The skill (`SKILL.md`) — orchestrator.
2. The agents (`agents/*.md`) — the seven roles the orchestrator delegates to.

Install both with one clone + two symlinks:

```bash
# 1. Clone.
mkdir -p ~/.claude/scheduled-tasks
git clone https://github.com/thanhtv368/sf-auto-agent.git ~/.claude/scheduled-tasks/sf-auto-agent

# 2. Link the agents into Claude's user-level agent directory so they resolve
#    by name from anywhere on the machine.
mkdir -p ~/.claude/agents
for f in ~/.claude/scheduled-tasks/sf-auto-agent/agents/*.md; do
  ln -sf "$f" ~/.claude/agents/
done
```

Claude Code discovers `SKILL.md` under `~/.claude/scheduled-tasks/` and any agent definition under `~/.claude/agents/`. Verify:

- `/sf-auto-agent` autocompletes.
- `Agent(type: "techlead", …)` and the other six names resolve.

### The seven agents

| Agent | When the orchestrator calls it | What it produces |
|-------|--------------------------------|------------------|
| `techlead` | After REPLIES_RECEIVED or for clear NEW records | Markdown implementation plan with Contract / Backend / Frontend / Test sections |
| `contract` | First step inside the implementation session | Schemas, types, DB migrations, empty procedure signatures |
| `backend` | After `contract` | Server-side logic implementing the plan |
| `frontend` | After `backend` | UI implementing the plan (skipped if no UI) |
| `tester` | After the implementation session opens a PR, and on every `ai-implemented` sweep when the PR head SHA changes | A `## 🧪 Test Report` PR comment with an embedded `**Tested commit:** \`<sha7>\`` line |
| `test-planner` | Called by `tester` when Playwright is configured | A JSON array of E2E flows to generate |
| `dev-fixer` | When the test sweep sees a failing report and the 3-attempt cap isn't hit | A minimal fix commit pushed to the PR branch + `[AI-AUTO-AGENT:FIX-APPLIED]` Chatter post |

### Tuning the agents per-project

The shipped agent prompts are intentionally generic (Node + TypeScript + tRPC-friendly, but not hard-coded). To tune for a specific stack, **override at the project level**: drop a file with the same name into `<project>/.claude/agents/`. Project-level agents shadow user-level ones for that project only.

Common overrides:

- Python project → swap "Zod schemas" for "Pydantic models" in `techlead.md` and `contract.md`.
- Repo with strict commit hygiene → add the commit message convention to `backend.md` and `dev-fixer.md`.
- Repo with non-Playwright E2E (Cypress, Vitest browser mode) → rewrite `tester.md`'s step 6.

---

## Step 3 — Create the global credentials file

```bash
cp ~/.claude/scheduled-tasks/sf-auto-agent/sf-auto-agent.env.template ~/.claude/sf-auto-agent.env
chmod 600 ~/.claude/sf-auto-agent.env
$EDITOR ~/.claude/sf-auto-agent.env
```

Fill in:

```
SF_LOGIN_URL=https://login.salesforce.com         # or test.salesforce.com for sandboxes
SF_CLIENT_ID=<consumer-key from step 1>
SF_CLIENT_SECRET=<consumer-secret from step 1>
SF_USERNAME=integration-user@example.com
SF_PASSWORD=<password><security-token>            # concatenated, no space
DEFAULT_GITHUB_REPO=org/repo                      # optional fallback
```

Smoke-test auth:

```bash
source ~/.claude/sf-auto-agent.env
curl -s -X POST "$SF_LOGIN_URL/services/oauth2/token" \
  -d "grant_type=password" \
  -d "client_id=$SF_CLIENT_ID" \
  -d "client_secret=$SF_CLIENT_SECRET" \
  -d "username=$SF_USERNAME" \
  --data-urlencode "password=$SF_PASSWORD" | jq
```

You should see `access_token`, `instance_url`, `id`. If you see `invalid_grant`, the most common causes are: wrong password+token concatenation, MFA on the user, or the connected app's "Permitted Users" policy.

---

## Step 4 — Configure a project

In the **root of each project** you want the agent to work on:

```bash
cd /path/to/your/project
mkdir -p .claude
cp ~/.claude/scheduled-tasks/sf-auto-agent/config.template.json .claude/sf-auto-agent.config.json
$EDITOR .claude/sf-auto-agent.config.json
```

A working example for a Node/Next.js project tracking Cases:

```json
{
  "projectName": "Commune",
  "github": { "repo": "thanhtv368/commune", "baseBranch": "main" },
  "salesforce": {
    "objectType": "Case",
    "recordKeyPrefix": "CASE",
    "summaryField": "Subject",
    "descriptionField": "Description",
    "ownerField": "OwnerId",
    "filterClause": "Status = 'In Progress' AND OwnerId = '{currentUserId}'",
    "activeImplementedFilter": "Status != 'Closed' AND Status != 'In Review'"
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

### Field-by-field

| Key | What it does |
|-----|--------------|
| `salesforce.objectType` | API name of the record type — `Case`, `Task`, `Work_Item__c`, etc. |
| `salesforce.summaryField` | Field used as a one-line title (in PR titles, branch names). |
| `salesforce.descriptionField` | Field fed to the techlead agent for planning. |
| `salesforce.ownerField` | Field compared to `{currentUserId}` in the filter. |
| `salesforce.filterClause` | SOQL `WHERE` clause (no leading `WHERE`). Use `{currentUserId}` literally; the skill substitutes it. |
| `salesforce.activeImplementedFilter` | The "not yet shipped" filter used by the test sweep. Tune per project's status values. |
| `stack.*Cmd` | Commands the tester runs in a fresh worktree. Adjust for non-Node projects (e.g. `pytest`, `go test`, `cargo build`). |
| `github.repo` | Where PRs are opened. The current `gh` auth must be able to push there. |

### Using a custom object instead of Case

Set `objectType` to its API name (e.g. `Work_Item__c`), then update `summaryField` / `descriptionField` to its actual field names. Everything else carries over.

---

## Step 5 — Enable Topics on the object

Topics are how the agent tracks state. They are off by default for custom objects.

In Salesforce: **Setup** → **Topics for Objects** → pick your object → **Enable Topics** → select the text fields you want indexed → Save.

For `Case`, Topics are typically already enabled. For `Work_Item__c` you'll need to enable them once.

---

## Step 6 — First run

```bash
cd /path/to/your/project
claude
```

In the Claude session, type:

```
/sf-auto-agent
```

What "good" looks like on first run:

- A SOQL probe is logged.
- If you have 0 matching records: log line `No actionable records` and the run ends.
- If you have one: the skill explores the codebase, decides if questions or a plan are appropriate, posts a Chatter comment on the record, and adds a Topic.

Check `.claude/sf-auto-agent-logs/orchestrator.log` for the timestamped summary line.

### Walking through a record end-to-end

Create a Case in Salesforce with a clear, scoped description (e.g. "Add a `/health` endpoint that returns 200"). Assign it to the integration user and set Status = `In Progress`. Then:

1. Run `/sf-auto-agent`. The agent posts a Chatter comment starting `[AI-AUTO-AGENT:PLAN]` and adds Topic `ai-plan-posted`.
2. Reply on the Case in Salesforce (Chatter) with the word `approved`.
3. Run `/sf-auto-agent` again. It removes `ai-plan-posted`, adds `ai-in-progress`, and spawns a background Claude session that pushes a branch and opens a PR.
4. Run `/sf-auto-agent` again ~5 minutes later. It detects the PR, adds `ai-implemented`, and spawns the tester. A `## 🧪 Test Report` comment lands on the PR with a `**Tested commit:** \`<sha7>\`` line.
5. If checks failed, the next run spawns the dev-fixer, which pushes a fix commit and the cycle continues (up to 3 fix attempts before the agent gives up and adds `ai-needs-human`).

---

## Step 7 — Schedule it

Pick one. All three work; they trade off where state lives.

### Option A: Claude's `/schedule` (recommended)

Runs even when your laptop is closed (managed by Anthropic).

```
/schedule create
```
- **Name**: `sf-auto-agent-<project>`
- **Cron**: `0 */2 * * *`  (every two hours)
- **Working directory**: the project root (so the per-project config loads)
- **Prompt**: `/sf-auto-agent`

Manage with `/schedule list`, `/schedule run`, `/schedule delete`.

### Option B: `/loop` inside an open session

```
/loop 2h /sf-auto-agent
```

Only runs while that Claude session is alive. Good for iteration; bad for production.

### Option C: System cron (headless)

```cron
0 */2 * * * cd /path/to/project && /usr/local/bin/claude --dangerously-skip-permissions -p "/sf-auto-agent" >> /path/to/project/.claude/sf-auto-agent-logs/cron.log 2>&1
```

No Claude UI required. Make sure `claude`, `gh`, `git`, `jq`, and `curl` are all on the cron PATH.

---

## How it talks to Salesforce

| Concept (Jira) | Concept (Salesforce) | API used |
|----------------|----------------------|----------|
| Ticket | Record (`Case` / custom object) | SOQL via `/services/data/v60.0/query` |
| JQL filter | SOQL `WHERE` clause | same |
| Comment | Chatter `FeedItem` | `/services/data/v60.0/chatter/feed-elements` |
| Label | Topic via `TopicAssignment` | `/services/data/v60.0/connect/topics/topic-assignments` |
| Status transition | Field update on the record | `PATCH /sobjects/<Object>/<Id>` |
| Auth | Basic + API token | OAuth 2.0 password flow |

All helpers live in `SKILL.md` under "Salesforce REST helpers".

---

## Topic vocabulary

The skill writes and reads these Topic names. Most are created on first use; pre-create them only if your org locks down topic creation.

| Topic | Meaning |
|-------|---------|
| `ai-questions` | Agent posted clarifying questions; awaiting human reply |
| `ai-plan-posted` | Plan posted; awaiting human approval |
| `ai-in-progress` | Implementation session running in the background |
| `ai-implemented` | PR opened — eligible for the test sweep |
| `ai-error` | Implementation session crashed; needs investigation |
| `ai-needs-human` | Fix loop hit the 3-attempt cap |

---

## Troubleshooting

**`invalid_grant` on auth**
Password+security-token must be concatenated with no separator. Reset the token after changing the password — the old token becomes invalid the moment you save a new password.

**`INSUFFICIENT_ACCESS_OR_READONLY` on TopicAssignment**
The integration user needs `Assign Topics` + `Create Topics` on the object. Enable in **Setup → Topics for Objects**.

**`MALFORMED_QUERY` from the SOQL probe**
Your `filterClause` references a field that doesn't exist on the object. Verify field API names (custom fields end in `__c`).

**Implementation session never finishes**
Tail `.claude/sf-auto-agent-logs/<RecordName>.log`. If it died mid-flow, the worktree at `.claude/worktrees/<branch>` is left for forensics — clean it up manually with `git worktree remove`.

**Tester loops re-testing the same commit**
Open the PR's latest test-report comment and confirm it has a `**Tested commit:** \`<sha7>\`` line. If missing, an old report from before this skill was upgraded won't match the current SHA and the sweep will keep re-testing. Delete the malformed comment.

**3-attempt fix cap reached**
The record now has `ai-needs-human`. Drop the topic manually after fixing the underlying issue to re-enable the loop.

---

## Layout reference

```
~/.claude/sf-auto-agent.env                     # global credentials (chmod 600)
~/.claude/scheduled-tasks/sf-auto-agent/
  SKILL.md                                      # the orchestrator
  sf-auto-agent.env.template                    # creds template
  config.template.json                          # per-project config template
  agents/                                       # the seven sub-agents
    techlead.md
    contract.md
    backend.md
    frontend.md
    tester.md
    dev-fixer.md
    test-planner.md
  README.md                                     # this file
~/.claude/agents/                               # symlinks to the seven above
  techlead.md -> …
  …

<project>/.claude/
  sf-auto-agent.config.json                     # per-project config (you create this)
  agents/                                       # OPTIONAL per-project overrides
    techlead.md                                 # shadows the user-level one
  sf-auto-agent-logs/                           # auto-created
    orchestrator.log                            # one summary line per run
    <RecordName>.log                            # per-record implementation log
    <RecordName>-test.log                       # per-record tester log
    <RecordName>-fix.log                        # per-record dev-fixer log
  worktrees/<branch>/                           # ephemeral, cleaned up on success
```

---

## License

MIT — do whatever, no warranty.
