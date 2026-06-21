# sf-auto-agent

A project-agnostic Claude Code skill that turns **Jira tickets into Salesforce PRs**.

It scans your team's "In Progress" Jira tickets, asks clarifying questions, posts an implementation plan, waits for human approval, then spawns background Claude sessions that build the Apex / LWC / metadata, validate against a per-branch scratch org, run Apex + LWC Jest tests, open a PR, post a test report, and self-heal failing checks — all while you sleep.

**This is for teams building Salesforce apps (SFDX projects) whose work is tracked in Jira.**

---

## What you get

After setup, every scheduled run will:

1. Query Jira for tickets assigned to you in "In Progress".
2. Classify each by **label** (`ai-questions`, `ai-plan-posted`, `ai-in-progress`, `ai-implemented`, `ai-needs-human`) or by Jira-comment markers.
3. Take the next correct action:
   - **NEW** → ask questions (if ambiguous) or post a plan.
   - **REPLIES_RECEIVED** → re-plan with the human's answers.
   - **APPROVED** → spawn a background implementation session: scratch org → contract metadata → Apex/Flow/Trigger → LWC/Aura → tests → PR.
   - **IN_PROGRESS** → check if the PR is up; if yes, mark `ai-implemented`.
   - **IMPLEMENTED w/ open PR** → run a tester (scanner, deploy validate, Apex tests + coverage, Jest), post a test report; if failing, run a dev-fixer (up to 3 attempts).
   - **MERGED** → clean up local logs.
4. Log a summary to `.claude/sf-auto-agent-logs/orchestrator.log`.

Labels are the source of truth — every state transition adds/removes one.

---

## Prerequisites

| Tool | Why |
|------|-----|
| Claude Code CLI | Hosts the skill and spawns sub-sessions |
| `sf` (Salesforce CLI v2) | Scratch orgs, deploy/validate, Apex tests |
| `gh` (GitHub CLI), authenticated | Opens PRs, posts test-report comments |
| `git` ≥ 2.5 | Worktrees |
| `jq` + `curl` | All Jira REST calls |
| Node.js + `npm` | LWC Jest, ESLint |
| Jira Cloud account with API token | Task tracking |
| Salesforce DevHub authenticated as `DevHub` (or your alias) | Source of scratch orgs |

Optional but recommended:

- `@salesforce/sfdx-scanner` installed as a plugin (`sf plugins install @salesforce/sfdx-scanner`) for Apex/Aura/LWC linting.
- `sfdx-lwc-jest` configured in `package.json` for LWC unit tests.

---

## Step 1 — Atlassian API token

In Jira: profile → **Account settings** → **Security** → **Create and manage API tokens** → **Create API token**.

Copy it. You'll paste it into `~/.claude/sf-auto-agent.env` in step 3.

---

## Step 2 — Authenticate a Salesforce DevHub locally

The implementation pipeline creates one scratch org per branch — it needs a DevHub to issue them.

```bash
sf org login web --alias DevHub --set-default-dev-hub
```

A browser opens; sign into your production org (or a Dev Hub-enabled trial). Verify:

```bash
sf org list
# DevHub should be listed with (D) marking it as the default Dev Hub.
```

If your org doesn't have a DevHub enabled: **Setup → Dev Hub → Enable Dev Hub**.

---

## Step 3 — Install the skill and the agents

The repo ships two things Claude needs:

1. The skill (`SKILL.md`) — orchestrator.
2. The agents (`agents/*.md`) — the seven roles the orchestrator delegates to.

Install both with one clone + a symlink loop:

```bash
mkdir -p ~/.claude/scheduled-tasks
git clone https://github.com/thanhtv368/sf-auto-agent.git ~/.claude/scheduled-tasks/sf-auto-agent

mkdir -p ~/.claude/agents
for f in ~/.claude/scheduled-tasks/sf-auto-agent/agents/*.md; do
  ln -sf "$f" ~/.claude/agents/
done
```

Verify: `/sf-auto-agent` autocompletes, and `Agent(type: "techlead", …)` resolves.

### The seven agents

| Agent | When the orchestrator calls it | What it produces |
|-------|--------------------------------|------------------|
| `techlead` | After REPLIES_RECEIVED or for clear NEW tickets | Markdown plan with Affected Metadata Surface / Contract / Backend / Frontend / Test sections |
| `contract` | First step inside the implementation session | Object + field XML, Apex interfaces, LWC public-API scaffolds, Permission Set grants |
| `backend` | After `contract` | Apex classes, triggers, Flows (bulkified, `with sharing`) |
| `frontend` | After `backend` | LWC (and rarely Aura) components, wired to `@AuraEnabled` methods |
| `tester` | After the implementation session opens a PR, and on every `ai-implemented` sweep when the PR head SHA changes | A `## 🧪 Test Report` PR comment with scanner, deploy-validate, Apex test, coverage, and Jest results — pinned to a `**Tested commit:** \`<sha7>\`` line |
| `test-planner` | Called by tester/dev-fixer when planning new Apex/Jest scenarios | A JSON object of `{apex, jest, utam}` scenarios to add |
| `dev-fixer` | When the test sweep sees a failing report and the 3-attempt cap isn't hit | A minimal fix commit pushed to the PR branch + `[AI-AUTO-AGENT:FIX-APPLIED]` Jira comment |

### Tuning per project

To override an agent for a single project, drop a same-named file in `<project>/.claude/agents/`. Project-level shadows user-level. Common overrides:

- Repo using **fflib** → add fflib-specific guidance to `backend.md`.
- Org with **strict trigger framework** → encode the dispatch pattern in `techlead.md` and `backend.md`.
- Repo with **UTAM** page objects → enable UTAM scenarios in `test-planner.md` and add an `npx utam-cli generate` step to `tester.md`.

---

## Step 4 — Create the global credentials file

```bash
cp ~/.claude/scheduled-tasks/sf-auto-agent/sf-auto-agent.env.template ~/.claude/sf-auto-agent.env
chmod 600 ~/.claude/sf-auto-agent.env
$EDITOR ~/.claude/sf-auto-agent.env
```

Fill in:

```
JIRA_BASE_URL=https://your-tenant.atlassian.net
JIRA_EMAIL=you@example.com
JIRA_API_TOKEN=<token from step 1>
SF_DEVHUB_ALIAS=DevHub
DEFAULT_GITHUB_REPO=org/repo
```

Smoke-test Jira auth:

```bash
source ~/.claude/sf-auto-agent.env
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "$JIRA_BASE_URL/rest/api/3/myself" | jq .emailAddress
```
You should see your email echoed back. `401` means wrong token or email; `404` means wrong base URL.

---

## Step 5 — Configure a project

In the **root of each SFDX repo** the agent should work on:

```bash
cd /path/to/your/sfdx/project
mkdir -p .claude
cp ~/.claude/scheduled-tasks/sf-auto-agent/config.template.json .claude/sf-auto-agent.config.json
$EDITOR .claude/sf-auto-agent.config.json
```

A working example:

```json
{
  "projectName": "AcmeBilling",
  "github": { "repo": "acme/acme-billing-sfdx", "baseBranch": "main" },
  "jira": {
    "projectKey": "BILL",
    "jqlFilter": "project=BILL AND status=\"In Progress\" AND assignee=currentUser()",
    "activeImplementedJql": "project=BILL AND labels=\"ai-implemented\" AND status NOT IN (\"Done\", \"In Review\")"
  },
  "salesforce": {
    "devhubAlias": "DevHub",
    "scratchOrgDefinition": "config/project-scratch-def.json",
    "apexCoverageThreshold": 80,
    "sourceDir": "force-app"
  },
  "stack": {
    "installCmd": "npm install",
    "lintCmd": "sf scanner run --target force-app --format table",
    "lwcLintCmd": "npm run lint",
    "validateCmd": "sf project deploy validate --source-dir force-app",
    "apexTestCmd": "sf apex run test --code-coverage --result-format human --wait 30",
    "lwcTestCmd": "npm run test:unit"
  }
}
```

### Field-by-field

| Key | What it does |
|-----|--------------|
| `jira.projectKey` | The Jira project (e.g. `BILL` for `BILL-123`). |
| `jira.jqlFilter` | What tickets to scan. Use the standard `currentUser()` JQL function. |
| `jira.activeImplementedJql` | Sweep filter for `ai-implemented` tickets that are NOT yet shipped. Tune to your workflow's status values. |
| `salesforce.devhubAlias` | The `sf` CLI alias for your DevHub. |
| `salesforce.scratchOrgDefinition` | Path to the scratch-org JSON. Default `config/project-scratch-def.json`. |
| `salesforce.apexCoverageThreshold` | Minimum overall coverage (%). The tester treats lower coverage as a failure. |
| `salesforce.sourceDir` | The SFDX source root. Default `force-app`. |
| `stack.lintCmd` | PMD/SF Scanner. Empty string disables. |
| `stack.lwcLintCmd` | Project's `package.json` lint script. |
| `stack.validateCmd` | `sf project deploy validate` — adds `--target-org <scratch>` automatically. |
| `stack.apexTestCmd` | `sf apex run test` — adds `--target-org <scratch>` automatically. |
| `stack.lwcTestCmd` | `sfdx-lwc-jest` runner. |

The tester will append `--target-org <branch-scratch>` to `validateCmd` and `apexTestCmd` — don't include it in the config.

---

## Step 6 — First run

```bash
cd /path/to/your/sfdx/project
claude
```
Inside the Claude session:

```
/sf-auto-agent
```

What "good" looks like on first run:

- A JQL probe and SOQL-free Jira fetch logs successfully.
- If you have 0 matching tickets: `No actionable tickets` and the run ends.
- If you have one: the agent explores `force-app/`, decides if questions or a plan are warranted, posts a Jira comment, and adds a label.

Check `.claude/sf-auto-agent-logs/orchestrator.log` for the timestamped summary.

### Walking through a ticket end-to-end

Create a Jira ticket with a clear description (e.g. *"Add a `Priority__c` picklist on `Case` and surface it in the `casePriorityBadge` LWC"*). Assign to yourself, set Status = `In Progress`. Then:

1. Run `/sf-auto-agent`. Agent posts `[AI-AUTO-AGENT:PLAN]` and adds label `ai-plan-posted`.
2. Reply on the ticket with `approved`.
3. Run `/sf-auto-agent` again. It removes `ai-plan-posted`, adds `ai-in-progress`, spawns a background session that:
   - Spins up `bill-123-scratch`.
   - Generates field metadata + Permission Set grant (contract agent).
   - Modifies the LWC + Apex (backend + frontend agents).
   - Deploys to the scratch org and runs Apex + Jest tests.
   - Opens a PR.
4. Run `/sf-auto-agent` again ~10 minutes later. It detects the PR, adds `ai-implemented`, and spawns the tester. A `## 🧪 Test Report` lands on the PR.
5. If anything failed, the next run spawns the dev-fixer (3-attempt cap).

---

## Step 7 — Schedule it

Pick one.

### Option A: `/schedule` (recommended)

```
/schedule create
```
- **Name**: `sf-auto-agent-<project>`
- **Cron**: `0 */2 * * *`
- **Working directory**: the project root
- **Prompt**: `/sf-auto-agent`

### Option B: `/loop` (open session)

```
/loop 2h /sf-auto-agent
```

### Option C: System cron (headless)

```cron
0 */2 * * * cd /path/to/project && /usr/local/bin/claude --dangerously-skip-permissions -p "/sf-auto-agent" >> /path/to/project/.claude/sf-auto-agent-logs/cron.log 2>&1
```
Make sure `claude`, `sf`, `gh`, `git`, `jq`, `curl` are all on the cron PATH.

---

## How it talks to Jira and Salesforce

| Concept | Where it lives | How the skill accesses it |
|---------|----------------|---------------------------|
| Ticket / state | Jira | `curl` against `/rest/api/3/search/jql`, `/issue/{KEY}/comment`, `/issue/{KEY}` (labels), `/issue/{KEY}/transitions` |
| State marker | Jira labels (primary) + comment markers (fallback) | `PUT /issue/{KEY}` `update.labels.add/remove` |
| Source code | GitHub | `gh` + `git worktree` |
| Scratch org | Salesforce DevHub | `sf org create/delete scratch` |
| Deploy + test | Scratch org | `sf project deploy validate / start`, `sf apex run test --code-coverage` |
| Lint | Local | `sf scanner run`, `npm run lint` |
| LWC unit | Local | `sfdx-lwc-jest` via `npm run test:unit` |

The skill never calls Salesforce's REST API directly — the `sf` CLI handles all org interactions, which means your existing org-auth and CI tooling continue to work.

---

## Label vocabulary

| Label | Meaning |
|-------|---------|
| `ai-questions` | Awaiting human reply on Jira |
| `ai-plan-posted` | Plan posted; awaiting human approval |
| `ai-in-progress` | Implementation session running |
| `ai-implemented` | PR opened — eligible for test sweep |
| `ai-error` | Implementation session crashed |
| `ai-needs-human` | Fix loop hit the 3-attempt cap |

---

## Troubleshooting

**Jira `401`** — Email + token combo wrong, or the token has expired.

**Jira `JQL error: "search" endpoint deprecated`** — Old skill code path. The current `SKILL.md` uses the new `POST /rest/api/3/search/jql`; make sure you pulled the latest.

**`sf org create scratch` fails with `OrgLimitExceeded`** — DevHub is out of scratch-org slots. List with `sf org list --all`, delete unused ones with `sf org delete scratch --target-org <alias>`. Or raise the limit in Setup → Dev Hub.

**Implementation session never finishes** — Tail `.claude/sf-auto-agent-logs/<TICKET>.log`. The worktree at `.claude/worktrees/<branch>` is left for forensics; `git worktree remove` it manually after fixing.

**Deploy validate fails with `INSUFFICIENT_ACCESS_OR_READONLY`** — The contract agent missed a Permission Set grant. Add the new field/object/class to the relevant `.permissionset-meta.xml` and re-run.

**Apex coverage below threshold** — The dev-fixer will pad coverage by adding real, asserting tests (no `Assert.areEqual(1, 1)` filler). If it keeps failing, the missing branch is unreachable; consider `@TestVisible` or refactor.

**Tester re-tests the same SHA forever** — The latest report comment is missing the `**Tested commit:** \`<sha7>\`` line. Delete it; the next run will post a fresh, well-formed one.

**3-attempt fix cap reached** — Ticket now has `ai-needs-human`. Remove the label after the human fix to re-enable the loop.

---

## Layout reference

```
~/.claude/sf-auto-agent.env                     # global credentials (chmod 600)
~/.claude/scheduled-tasks/sf-auto-agent/
  SKILL.md                                      # orchestrator
  sf-auto-agent.env.template
  config.template.json
  agents/
    techlead.md, contract.md, backend.md, frontend.md,
    tester.md, dev-fixer.md, test-planner.md
  README.md
~/.claude/agents/                               # symlinks to the seven above

<project>/.claude/
  sf-auto-agent.config.json                     # per-project config
  agents/                                       # OPTIONAL overrides
  sf-auto-agent-logs/                           # auto-created
    orchestrator.log
    <TICKET-KEY>.log
    <TICKET-KEY>-test.log
    <TICKET-KEY>-fix.log
  worktrees/<branch>/                           # ephemeral
```

---

## License

MIT — do whatever, no warranty.
