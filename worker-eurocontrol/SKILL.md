---
name: worker-eurocontrol
description: Eurocontrol-environment variant of worker — Claude is the only agent kind installed there (no OpenCode/Big Pickle/MiMo, no Grok, no Google Antigravity), and it's a single hardcoded board with no repo-name routing. Executes a kanban task by decomposing it across one or more Claude-only Herdr agents, verifies, presents for human approval, iterates, marks done. Use this instead of the regular worker skill whenever the task is on the Eurocontrol board.
---

# Worker — Eurocontrol Environment (Claude-Only)

This is the Eurocontrol-specific variant of the regular `worker` skill.
**Only `claude` is installed as an agent kind in this environment** — no
OpenCode (Big Pickle / MiMo V2.5), no Grok, no Google Antigravity.
Every agent dispatch in this skill uses `--kind claude`, full stop. Use the
regular `worker` skill for Guiñotazo / Universiteit Utrecht work; use this
one only when the task is on the Eurocontrol board.

**Cardinal rule: the human approves `done` — never the agent, never the
orchestrator. The card moves `review → done` only on explicit user approval.**

**Hard requirement — specialist agents never share the orchestrator's pane or
tab.** A subagent must never run as a `herdr pane split` off the
orchestrator's own tab. Every specialist agent gets its own brand-new tab
(`herdr tab create`), never a split pane, no exceptions. When that agent's
work is done and no longer needed, close its **entire tab**
(`herdr tab close <tab_id>`), not just its pane. See Phase 4 step 1 and
Phase 9 for the exact commands.

---

## Before You Begin — Context Hygiene

This skill is long and multi-phase, and Phase 4 can spin up multiple Claude
agents — it accumulates a lot of tokens in the main conversation. **The first
thing you do when this skill is invoked is remind the user to run `/clear`
first**, unless the conversation is already fresh. Say something like: "This
is a long-running skill — worth running `/clear` first to keep the session
light. Ready when you are." Then wait for them to clear and re-invoke, or to
explicitly say to proceed without clearing.

---

## Prerequisites

- `HERDR_ENV=1` (required — this skill runs inside Herdr). Herdr itself is
  installed — it's the *other agent kinds* (OpenCode, Grok, Google
  Antigravity) that are not installed in this environment, not Herdr.
- `~/.claude/skills/scripts/backlog` wrapper, with `BACKLOG_PROJECT="Eurocontrol"`
  exported before every call.
- Taskwarrior + taskwarrior-kanban backend (`~/.task`).
- Kanban board running at `http://127.0.0.1:8787/` (run `/setup_kanban_board`
  if not).

---

## Board (One Board, No Routing)

This skill always targets a single board:

```bash
export BACKLOG_PROJECT="Eurocontrol"
```

Unlike the regular `worker` skill (which reads both Guiñotazo and
Universiteit Utrecht and asks the user which one), there is nothing to
resolve here — every task this skill claims/reviews/completes is on the
Eurocontrol board. There is also **no `repo:` field to resolve** — this
environment has no multi-repo fan-out; operate in the current working
directory throughout.

---

## Agent Portfolio (Claude Only)

| Agent Label | Herdr Kind | Best For |
|-------------|-----------|----------|
| **Claude** | `claude` (Sonnet/Opus) | Every sub-task — architecture decisions, multi-file implementation, focused fixes, doc reading, verification. It's the only agent kind installed in this environment. |

**There is no fallback agent kind in this environment.** If a Claude sub-task
agent is out of credits or rate-limited, **stop that sub-task and tell the
user** — do not suggest Grok, Google Antigravity, or any OpenCode model
(Big Pickle/MiMo); none of them are installed here. Offer only, via
`AskUserQuestion`: `Retry the same sub-task later`, `Do this sub-task
yourself (no sub-agent)`, or `Skip this sub-task for now`.

**Effort selection**: before starting any Claude sub-task agent in Phase 4,
propose a recommended `--effort` level (`low`, `medium`, `high`, `xhigh`,
`max`) based on that sub-task's actual complexity, and confirm with the user
via `AskUserQuestion` before spawning — one question per sub-task agent,
batched into as few calls as fit the 4-question limit. Recommend `medium` for
a routine, well-scoped implementation sub-task; `high` for one with real
architecture or correctness risk; `xhigh`/`max` only for something Phase 3's
decomposition flagged as unusually high-risk.

Launch with the chosen level appended after `--`:
`herdr agent start <name> --kind claude --pane <id> -- --effort <level>`.

---

## Verification Command Protocol

Unlike the regular `worker` skill (which hardcodes a Terraform protocol for
Universiteit Utrecht and an `npm` command for Guiñotazo), this environment's
tech stack is whatever Phase 3's repo scan finds it to be — **do not assume
Terraform, do not assume npm.** Detect the actual verification command from
the repo's own tooling signals (`package.json`, `Makefile`, `requirements.txt`,
`pom.xml`, etc.) in Phase 3 and use that exact command in Phase 5.

**If any Verification Plan step on the card is itself flagged as
unsafe/expensive/needing elevated access the orchestrator doesn't hold**
(the card should say so explicitly per `kanban-task-eurocontrol`'s template):
hand that exact command to the user as plain text rather than running it
yourself, the same way the regular `worker` skill's Terraform protocol hands
`plan`/`apply` to the user — don't invoke it via Bash, don't poll for the
result, resume once the user reports back.

---

## Workflow

### Phase 0 — Ensure Board & Prerequisites

1. Verify `HERDR_ENV=1` and `~/.claude/skills/scripts/backlog` exists. Stop if not.
2. Ensure kanban board is up (`curl http://127.0.0.1:8787/` → 200). If not,
   run `/setup_kanban_board`.

### Phase 1 — Select the Task

```bash
export BACKLOG_PROJECT="Eurocontrol"
bash ~/.claude/skills/scripts/backlog next
```

**If the user didn't provide a task number**: Ask via `AskUserQuestion` with
one option per open card (show ID, priority, agent, first line of
description). High priority first.

**Capture the UUID** immediately:
```bash
bash ~/.claude/skills/scripts/backlog uuid <id>
```
(Numeric IDs recycle; all downstream commands must use the UUID.)

Read the full card description:
```bash
task <UUID> export
```
Parse out: title, description, acceptance criteria, verification command,
assigned agent (if any), priority.

### Phase 2 — Claim the Card (on behalf of the worker)

```bash
export BACKLOG_PROJECT="Eurocontrol"
bash ~/.claude/skills/scripts/backlog claim <UUID>
```
This moves it to `active` and annotates `claimed-by:worker`.

### Phase 3 — Deep Task Analysis & Decomposition (Internal)

**Read the repo**:
- Structure, stack, conventions, any project-specific rules
  (`AGENTS.md`/`CLAUDE.md`).
- Detect and run the project's actual verification command once (see
  Verification Command Protocol above) for a live baseline.
- Check `git log --oneline -20` for recent direction.

**Decompose the task** into sub-tasks. Since every sub-task lands on Claude
regardless of type, decomposition here is purely about splitting work into
independently-parallelizable pieces (e.g. "read/summarize the spec" vs.
"implement the change" vs. "cross-check the diff"), not about matching
sub-tasks to different agent strengths — there's only one agent to match to.

**Output**: A **decomposition plan** (internal) listing each sub-task, its
prompt template, and expected deliverable — every slot assigned to Claude.

### Phase 4 — Spawn Claude Sub-Agents (Herdr)

**First**, for every sub-task, run the Effort Selection step from the Agent
Portfolio section above via `AskUserQuestion` and get the user's confirmed
effort level before starting each agent.

For each sub-task in the decomposition plan:

1. **Never split a pane off the orchestrator's tab, and never reuse an idle
   pane — every specialist agent gets a brand-new tab of its own.**
   `herdr tab create` per agent, every time, no exceptions:
   ```bash
   TAB_JSON=$(herdr tab create --cwd "$PWD" --no-focus)
   TAB_ID=$(echo "$TAB_JSON" | jq -r '.result.tab.tab_id')
   PANE_ID=$(echo "$TAB_JSON" | jq -r '.result.root_pane.pane_id')
   herdr agent start worker-<subtask>-<n> --kind claude --pane "$PANE_ID" -- --effort <level>
   ```
   Track: `subtask_name → { agent_name, tab_id, pane_id, effort }`.

2. **Send the composed prompt** (self-contained, no "as discussed"
   references):
   - Repo path (current working directory) and the task UUID — tell the
     agent to run `task <UUID> export` itself to read the full card rather
     than pasting the card's full description into the prompt; that avoids
     re-typing a potentially-long card body into every sub-task's prompt.
   - Sub-task scope (exact files/functions to touch).
   - Acceptance criteria **for this sub-task specifically** (a distilled
     subset of the card's full criteria, not the whole list).
   - Verification command to run.
   - **Constraint**: "Stay in scope. Do not touch unrelated files. Report
     changes and any uncertainties. Do NOT run `backlog review/done`."

3. **Wait & collect**:
   ```bash
   herdr agent prompt <agent_name> "<prompt>" --wait --timeout 600000
   herdr agent read <agent_name> --source recent-unwrapped --lines 300
   ```

4. **If agent blocks** (`herdr agent get` shows `blocked`): inspect, ask user
   how to respond — never answer approvals on their behalf.

5. **If agent is out of credits / billing-failed**: **stop that sub-task. Do
   not auto-retry.** Tell the user which sub-task failed. **There is no
   fallback agent kind in this environment** — do not suggest Grok, Google
   Antigravity, or any OpenCode model. Offer via `AskUserQuestion`: `Retry
   the same sub-task later`, `Do this sub-task yourself (no sub-agent)`, or
   `Skip this sub-task for now`.

### Phase 5 — Synthesize & Verify (Orchestrator Work)

**After all sub-tasks complete:**

1. **Aggregate changes**: `git status --short`, `git diff` — verify only
   plausible files changed (scope of the card).
2. **Run full verification command** (from card / Phase 3) — the exact
   command detected there, not an assumed one. Must pass clean.
3. **Execute the card's `## Verification Plan` step-by-step** — every card
   authored via `kanban-task-eurocontrol` carries one; older cards may not.
   This is the authoritative "is it actually done" check, distinct from
   step 2's generic build/test pass: run each listed check in order and
   compare against its stated expected result. For any step the plan marks
   as unsafe/needing elevated access/user-only, hand that exact step to the
   user (see Verification Command Protocol above) — don't skip it silently
   and don't attempt a workaround. Record pass/fail (with actual command
   output) for every step — this becomes the Phase 7 evidence.
   - **If the card has no `## Verification Plan` section**: derive an
     equivalent ad-hoc checklist yourself from its Acceptance Criteria before
     proceeding, and say explicitly in Phase 7 that this was synthesized.
4. **Check acceptance criteria** from the card — each must be visibly
   satisfied by the changes (cross-reference against step 3's results).
5. **If any check fails**: Loop back to Phase 4 for the relevant sub-task(s)
   with a fix prompt (specific failure output). Max 2 fix cycles per
   sub-task.

### Phase 6 — Move Card to REVIEW

```bash
export BACKLOG_PROJECT="Eurocontrol"
bash ~/.claude/skills/scripts/backlog review <UUID>
```
Card is now in the board's REVIEW column. **Do NOT run `backlog done`.**

### Phase 7 — Human Verification Gate (Mandatory)

Present to the user:
- Summary of what each Claude sub-agent did (files changed, key decisions).
- Verification evidence: test count (N/N), lint clean, build success.
- **Verification Plan results**: each step from the card's `## Verification
  Plan` (or the synthesized ad-hoc checklist) with ✓/✗ and the actual
  output/value observed.
- Acceptance criteria checklist (each ✓/✗, cross-referenced to the
  Verification Plan step(s) that proved it).
- Any uncertainties or items the agents flagged.

**Ask via `AskUserQuestion`:**

1. **Verdict**: `Approve — mark done` | `Let me test first` | `Needs
   changes: <feedback>`
2. If "Let me test first": offer to start the app and give the URL/command.
   Wait for user to return, then re-ask.
3. If "Needs changes": capture exact feedback.

### Phase 8 — Send-Back Loop (on rejection, max 2 cycles)

On rejection:
1. Analyze feedback + your own findings → create a focused fix prompt.
2. Dispatch to the **same Claude sub-agent** (it has working-tree context):
   ```bash
   herdr agent prompt <agent_name> "<feedback + specific fix request>" --wait --timeout 300000
   ```
3. Re-run Phase 5 (verify) + Phase 6 (review) + Phase 7 (ask again).
4. Cap at 2 cycles. On 3rd failure, stop and present full history to user
   for decision.

### Phase 9 — On Approval: Mark Done & Close Agents

**Only on explicit "Approve — mark done":**
```bash
export BACKLOG_PROJECT="Eurocontrol"
bash ~/.claude/skills/scripts/backlog done <UUID>
```

**Close every specialist agent's tab spawned in Phase 4** — since Phase 4
always opens a brand-new tab per agent, every tab tracked there is yours to
close **in full**, not just the pane inside it:
```bash
herdr tab close <tab_id>
```
- Do NOT close while card is still in `review` — only after `done`.

### Phase 10 — Final Report

Report to user:
- Task UUID, title, final state (`done`).
- Which Claude sub-agents participated (effort level, fresh vs reused).
- Summary of changes per sub-agent.
- Verification evidence (test count, lint, build, acceptance criteria).
- Human verdict + result (approved + done, or rejected + retries).
- Board URL: `http://127.0.0.1:8787/`.
- Optional: `herdr notification show "Worker" --body "Task <UUID> done"`.

---

## Usage as a Command

Invoke as `worker-eurocontrol` whenever the task is on the Eurocontrol
board — never the regular `worker` skill for this environment, since that
one assumes agent kinds (OpenCode/Grok/Antigravity) that aren't installed
here. Interactive at Phase 1 (task pick), Phase 7 (verdict), and any
send-back.

---

## Notes for the Agent Running This Skill

- **Never guess file paths** — read them first (`read`/`grep`/`glob`).
- **Never assume tooling** — detect and run the project's actual verification
  command live (Phase 3); do not assume Terraform or npm.
- **Claude is the only agent kind here** — never dispatch OpenCode, Grok, or
  Google Antigravity; they are not installed in this environment. If a
  sub-task's Claude agent fails (credits/rate limit), there is no fallback —
  stop and tell the user.
- **Decomposition splits work, not agent strength** — every slot is Claude;
  decompose only to parallelize across Herdr tabs, not to match strengths.
- **Prompts must be self-contained** — sub-agents have zero shared context.
- **Every specialist agent gets its own brand-new tab, never a pane split off
  the orchestrator, never reused** — track every tab/agent from Phase 4 and
  close every tab in full (`herdr tab close`), only after `done`.
- **Two gates**: mechanical verification (Phase 5) + human approval
  (Phase 7).
- **Max 2 send-back cycles** — then escalate to user.
- **Anchor everything** — every claim about code must have `file:line` from
  your Phase 3 scan.
