---
name: worker
description: Execute a kanban task by orchestrating multiple specialist Herdr agents (Claude, Big Pickle, MiMo, Grok). Decomposes task, verifies, presents for human approval, iterates, marks done. Use when you have a task on the board and want it executed with multi-agent support.
---

# Worker — Execute a Kanban Task with Multi-Agent Herdr Support

Pull a task from the Taskwarrior kanban board, dispatch it to a coordinated team
of AI agents via Herdr CLI, verify the work against the card's acceptance
criteria, present for human review, iterate until approved, then close agents
and update the board.

**Cardinal rule: the human approves `done` — never the agent, never the
orchestrator. The card moves `review → done` only on explicit user approval.**

**Hard requirement — specialist agents never share the orchestrator's pane or
tab.** A subagent must never run as a `herdr pane split` off the orchestrator's
own tab — splitting shares screen space with whatever else is in that tab, and
as more agents pile in the view gets cramped and unreadable. Every specialist
agent gets its own brand-new tab (`herdr tab create`), never a split pane, no
exceptions. When that agent's work is done and no longer needed, close its
**entire tab** (`herdr tab close <tab_id>`), not just its pane. See Phase 4
step 1 and Phase 9 for the exact commands.

---

## Before You Begin — Context Hygiene

This skill is long and multi-phase, and Phase 4 can spin up multiple Herdr
agents — it accumulates a lot of tokens in the main conversation. **The first
thing you do when this skill is invoked is remind the user to run `/clear`
first**, unless the conversation is already fresh (first message of the
session, or the user just cleared). Say something like: "This is a
long-running skill — worth running `/clear` first to keep the session light.
Ready when you are." Then wait for them to clear and re-invoke, or to
explicitly say to proceed without clearing — don't start Phase 0 in the
meantime.

---

**Eurocontrol board work uses a different skill entirely** —
`worker-eurocontrol`, not this one. That environment only has `claude`
installed as an agent kind (no OpenCode/Grok/Antigravity), and is a single
hardcoded board with no repo routing, so the dedicated skill drops all of
that machinery instead of trying to degrade gracefully here.

## Prerequisites

- `HERDR_ENV=1` (required — this skill runs inside Herdr).
- `~/.claude/skills/scripts/backlog` wrapper (project-agnostic; scopes every call via
  `BACKLOG_PROJECT`, see Board Routing below).
- Taskwarrior + taskwarrior-kanban backend (`~/.task`, shared across both
  boards — they're lanes on the same server, distinguished by Taskwarrior's
  `project:` field).
- Kanban board running at `http://127.0.0.1:8787/` (run `/setup_kanban_board` if not).

---

## Board Routing (Two Boards)

The shared taskwarrior-kanban server hosts **two** boards, selected via
`BACKLOG_PROJECT`:

| `BACKLOG_PROJECT` | Board name |
|--------------------|------------|
| `guiñotazo` | Guiñotazo |
| `Universiteit Utrecht` | Universiteit Utrecht |

Unlike `kanban-task`'s push step (which can infer the board from the repo
name), **`worker` never auto-picks a board when selecting a task to pull.**
The working directory doesn't reliably say which board's card is intended on
any given run, so: **always read both boards first, then ask the user which
one** before listing individual cards (Phase 1). Never default silently to
Guiñotazo just because it's the original board.

Carry the resolved `BACKLOG_PROJECT` forward through the rest of the workflow
(claim, review, done all need the same value the card was read under).

---

## Agent Portfolio (Herdr Kinds)

**See [`../shared/agent-routing.md`](../shared/agent-routing.md) for the full
routing table, OpenCode invocation shape, effort selection, and
billing-failure fallback handling — read it before dispatching any agent in
this skill.** The table's "Best-fit Agent" column is what Phase 3's
decomposition table below maps sub-task types onto.

**Strategy**: The worker agent (this skill) acts as the **orchestrator** — it
decomposes the task, writes focused prompts for each specialist agent, collects
results, synthesizes, and verifies. Specialist agents do the hands-on work.
Unlike `kanban-task`'s proposal phase, sub-tasks here are actual
implementation work, so OpenCode-kind agents use whichever persona
(`build`/`plan`) the shared table's row specifies — there's no
proposal-only override forcing everything to `plan`.

---

## Terraform Plan/Apply Protocol (Universiteit Utrecht board only)

Tasks on the **Universiteit Utrecht** board are Terraform/Azure changes, not
npm/test-suite repos, and every command below runs from `REPO_PATH` (the
card's `repo` field, resolved in Phase 1) — never assume the current
directory already is the right checkout. Running `terraform plan`/`terraform apply` directly
from this session has repeatedly hit two problems: large remote-state
downloads can intermittently truncate over this sandbox's network path (looks
like state corruption but usually isn't — retries or a later attempt often
succeed), and these commands routinely run long enough to blow past the
default tool timeout on a repo this size, forcing pointless background-task
juggling. **Never run `terraform plan` or `terraform apply` yourself for a
Universiteit Utrecht task — hand the command to the user instead:**

1. Compose the exact command. For a `plan`, always save it to a scratch path
   under `/tmp` (not the repo) with `-out=`, so the user can `apply` the exact
   reviewed plan afterward instead of a fresh, possibly-different one:
   ```bash
   terraform plan -var-file=<environment>.tfvars -out=/tmp/tf-plan-<task-id>.tfplan
   terraform apply /tmp/tf-plan-<task-id>.tfplan
   ```
   If no saved plan is being reused, a direct `terraform apply
   -var-file=<environment>.tfvars` is fine too — just say which form you mean
   and why (e.g. "no plan file yet, so this will compute its own plan and ask
   you to confirm").
2. **Paste the exact, ready-to-run command as plain text** — don't invoke it
   via Bash yourself, and don't summarize it in prose the user has to
   reconstruct.
3. Say plainly that you're waiting for them to run it and report back, and
   that you'll pick the workflow back up automatically once they do — don't
   poll for the result and don't retry the command yourself in a loop.
4. When they paste back the result (success, error, or a partial-apply
   report), resume from there: `terraform fmt`/`terraform validate` are fast,
   local, and safe, so keep running those yourself directly; reconcile
   whatever plan/apply output the user gave you against the card's acceptance
   criteria, and continue to the next phase.

This protocol is scoped to Universiteit Utrecht board tasks only. Guiñotazo
tasks keep using `npm run lint && npm run test && npm run build`, run
directly by you, exactly as elsewhere in this skill.

---

## Workflow

### Phase 0 — Ensure Board & Prerequisites

1. Verify `HERDR_ENV=1` and `~/.claude/skills/scripts/backlog` exists. Stop if not.
2. Ensure kanban board is up (`curl http://127.0.0.1:8787/` → 200). If not, run
   `/setup_kanban_board`.

### Phase 1 — Select the Board, Then the Task

**Unless the user already told you which board** (e.g. named the repo, said
"Guiñotazo"/"Utrecht" outright, or gave a UUID that only exists on one board),
resolve the board before anything else:

1. **Read both boards** so the question in step 2 is informed, not a blind
   guess:
   ```bash
   BACKLOG_PROJECT=guiñotazo bash ~/.claude/skills/scripts/backlog next
   BACKLOG_PROJECT="Universiteit Utrecht" bash ~/.claude/skills/scripts/backlog next
   ```
2. **Ask via `AskUserQuestion`** which board to pull from — label each option
   with the board name and a quick count/summary of what you just read (e.g.
   `Guiñotazo — 3 open (2 high priority)` / `Universiteit Utrecht — 1 open`).
   Do not skip this even if one board looks empty or one obviously matches the
   current directory — ask every time it's ambiguous.

Once the board is resolved, keep `BACKLOG_PROJECT` set for **every** command
below (claim/review/done all need the same value the card was read under).

**Select the task** from the chosen board's list (already fetched in step 1,
or re-run `next` if it's gone stale):

**If the user didn't provide a task number**: Ask via `AskUserQuestion` with
one option per open card (show ID, priority, agent, first line of description).
High priority first.

**Capture the UUID** immediately:
```bash
BACKLOG_PROJECT=<resolved> bash ~/.claude/skills/scripts/backlog uuid <id>
```
(Numeric IDs recycle; all downstream commands must use the UUID.)

Read the full card description:
```bash
task <UUID> export
```
Parse out: title, description, acceptance criteria, verification command,
assigned agent (if any), priority, and (Universiteit Utrecht only) `repo`.

**If the resolved board is Universiteit Utrecht, the card's `repo` field is
mandatory** — that board fans out across every `tf-*` repo in the home
directory, and there is no reliable way to guess which one from the
orchestrator's own cwd. Resolve it explicitly:
```bash
BACKLOG_PROJECT=<resolved> bash ~/.claude/skills/scripts/backlog repo <UUID>
```
- **If it prints a path**: `cd` there (or otherwise track it as `REPO_PATH`)
  and use it for every subsequent phase — Phase 3's repo scan, Phase 4's agent
  `--cwd`, and the Terraform Plan/Apply Protocol commands all operate in
  `REPO_PATH`, not wherever this session happened to start.
- **If it errors ("no repo: set")**: this card predates the repo requirement.
  Stop and ask the user for the absolute repo path via `AskUserQuestion`
  before proceeding — don't guess from the current directory. Once given,
  backfill it onto the card so future runs don't hit this again:
  ```bash
  BACKLOG_PROJECT=<resolved> bash ~/.claude/skills/scripts/backlog repo <UUID> <path>
  ```
Guiñotazo cards have no such requirement — operate in the current repo as
before.

### Phase 2 — Claim the Card (on behalf of the worker)

```bash
BACKLOG_PROJECT=<resolved> bash ~/.claude/skills/scripts/backlog claim <UUID>
```
This moves it to `active` and annotates `claimed-by:worker`.

### Phase 3 — Deep Task Analysis & Agent Decomposition (Internal)

**Read the repo** (same depth as `kanban-task` Phase 1). For a Universiteit
Utrecht card this means `REPO_PATH` resolved in Phase 1 — `cd "$REPO_PATH"`
(or prefix every command with it) before doing any of the following, never
the orchestrator's own starting directory:
- Structure, stack, conventions, engine purity rules (`AGENTS.md`).
- Run verification command once (`npm run lint && npm run test && npm run build`)
  for live baseline.
- Check `git log --oneline -20` for recent direction.

**Decompose the task** into sub-tasks matched to agent strengths:

| Sub-task Type | Agent | Example Prompt Focus |
|---------------|-------|---------------------|
| Architecture / high-level design | MiMo V2.5 | "Given this task and these constraints, propose a file-level plan with trade-offs" |
| Complex implementation (multi-file) | Big Pickle / Claude | "Implement X in files A, B, C following pattern at Y. Run tests." |
| Focused edits / bug fixes | Big Pickle | "Fix the issue at file.ts:NN per acceptance criteria Z" |
| Long doc / spec reading | Big Pickle | "Read these 5 files and summarize the current pattern for X" |
| Quick verification / lint | Big Pickle | "Run lint on these changed files and report errors" |
| Reasoning / trade-off analysis | MiMo V2.5 | "Compare approach A vs B for this task given constraints" |
| Independent cross-check on a completed sub-task (when both other agents used would be opencode-kind) | Grok | "Review this diff for X against the acceptance criteria; flag anything the implementer may have missed" |

**Output**: A **decomposition plan** (internal) listing each sub-task, assigned
agent, prompt template, and expected deliverable.

### Phase 4 — Spawn Specialist Agents (Herdr)

**First**, for every sub-task landing on Claude or Grok (per the decomposition
plan), run the **Effort Selection** step from the Agent Portfolio section
above via `AskUserQuestion` and get the user's confirmed effort level before
starting those agents. OpenCode-kind sub-tasks skip this.

For each sub-task in the decomposition plan:

1. **Never split a pane off the orchestrator's tab, and never reuse an idle
   pane — every specialist agent gets a brand-new tab of its own.** A pane
   split shares screen space with the orchestrator's own view (and any other
   agent already in that tab), which is exactly the "can't see what's going
   on" problem — it gets cramped and cluttered as agents pile in. An
   idle-looking pane may also belong to the user's own session or another
   task's agent; reusing it risks clobbering work-in-progress or mixing two
   agents' context together. `herdr tab create` per agent, every time, no
   exceptions:
   ```bash
   # Create a brand-new tab (its own top-level view, not a split) for this agent.
   # --cwd MUST be REPO_PATH (Phase 1's resolved repo) for a Universiteit
   # Utrecht card, NOT the orchestrator's own $PWD — that's how the card's
   # repo: field actually reaches the specialist agent's working directory.
   TAB_JSON=$(herdr tab create --cwd "${REPO_PATH:-$PWD}" --no-focus)
   TAB_ID=$(echo "$TAB_JSON" | jq -r '.result.tab.tab_id')
   PANE_ID=$(echo "$TAB_JSON" | jq -r '.result.root_pane.pane_id')
   # Start agent — the model/agent/effort go AFTER "--" as the underlying
   # binary's own CLI args, not as herdr-level flags:
   herdr agent start worker-<subtask>-<n> --kind claude --pane "$PANE_ID" -- --effort <level>
   herdr agent start worker-<subtask>-<n> --kind grok --pane "$PANE_ID" -- --effort <level>
   herdr agent start worker-<subtask>-<n> --kind opencode --pane "$PANE_ID" -- -m opencode/<model> --agent <build|plan>
   herdr agent start worker-<subtask>-<n> --kind agy --pane "$PANE_ID" -- --model <model-name> --effort <level>
   ```
   Track: `subtask_name → { agent_name, tab_id, pane_id, kind, model, effort (Claude/Grok/Antigravity only) }`.
   Whichever kind is used for this sub-task, explicitly say so to the user
   before starting it (e.g. "dispatching Big Pickle for the implementation
   sub-task" / "dispatching Grok for the cross-check on X").

2. **Send the composed prompt** (self-contained, no "as discussed" references):
   - Repo path (`REPO_PATH` from Phase 1 for Universiteit Utrecht, the
     current repo otherwise) and the task UUID — tell the agent to run
     `task <UUID> export` itself to read the full card (it has the same
     `~/.task` access this session does) rather than pasting the card's
     full description into the prompt; that avoids re-typing a
     potentially-long card body into every sub-task's prompt.
   - Sub-task scope (exact files/functions to touch).
   - Acceptance criteria **for this sub-task specifically** (a distilled
     subset of the card's full criteria, not the whole list — that's still
     worth composing by hand since not every sub-task owns every criterion).
   - Verification command to run.
   - **Constraint**: "Stay in scope. Do not touch unrelated files. Report changes
     and any uncertainties. Do NOT run `backlog review/done`."

3. **Wait & collect** (`--wait --timeout 300000` or longer for complex work):
   ```bash
   herdr agent prompt <agent_name> "<prompt>" --wait --timeout 600000
   herdr agent read <agent_name> --source recent-unwrapped --lines 300
   ```

4. **If agent blocks** (`herdr agent get` shows `blocked`): inspect, ask user
   how to respond — never answer approvals on their behalf.

5. **If agent is out of credits / billing-failed**: stop that sub-task —
   follow `../shared/agent-routing.md`'s "Billing-Failure / Rate-Limit
   Fallback" section, substituting "sub-task" for "proposal slot." Resume
   Phase 4 for that sub-task only once the user picks.

### Phase 5 — Synthesize & Verify (Orchestrator Work)

**After all sub-tasks complete:**

1. **Aggregate changes**: `git status --short`, `git diff` — verify only
   plausible files changed (scope of the card).
2. **Run full verification command** (from card / Phase 1):
   - Universiteit Utrecht board: `terraform fmt -check` and `terraform
     validate` directly (fast, local, safe). For `terraform plan`/`apply`,
     follow the **Terraform Plan/Apply Protocol** above — hand the user the
     command, wait for their result, don't run it yourself.
   - Guiñotazo board:
     ```bash
     npm run lint && npm run test && npm run build
     ```
   Must pass clean.
3. **Execute the card's `## Verification Plan` step-by-step** — every card
   authored via `kanban-task` carries one; older cards may not. This is the
   authoritative "is it actually done" check, distinct from step 2's generic
   build/test pass: run each listed check in order and compare against its
   stated expected result, not just "no error was thrown". For any step the
   plan marks as unsafe/needing elevated access/user-only (e.g. `terraform
   plan/apply` under the Terraform Plan/Apply Protocol, or anything needing
   RBAC/IAM the orchestrator doesn't hold), hand that exact step to the user
   the same way the Terraform protocol does — don't skip it silently and don't
   attempt a workaround. Record pass/fail (with the actual command output, not
   just a verdict) for every step — this becomes the Phase 7 evidence.
   - **If the card has no `## Verification Plan` section**: derive an
     equivalent ad-hoc checklist yourself from its Acceptance Criteria before
     proceeding — one concrete check per criterion — and say explicitly in
     Phase 7 that this was synthesized because the card predates the
     verification-plan requirement.
4. **Check acceptance criteria** from the card — each must be visibly satisfied
   by the changes (cross-reference against step 3's results; every criterion
   should trace to at least one verification-plan step that passed).
5. **Engine purity check**: Ensure no `src/engine` mutations (only additive
   exports via `src/engine/index.ts`).
6. **If any check fails**: Loop back to Phase 4 for the relevant sub-task(s)
   with a fix prompt (specific failure output). Max 2 fix cycles per sub-task.

### Phase 6 — Move Card to REVIEW

```bash
BACKLOG_PROJECT=<resolved> bash ~/.claude/skills/scripts/backlog review <UUID>
```
Card is now in the board's REVIEW column. **Do NOT run `backlog done`.**

### Phase 7 — Human Verification Gate (Mandatory)

Present to the user:
- Summary of what each agent did (files changed, key decisions).
- Verification evidence: test count (N/N), lint clean, build success.
- **Verification Plan results**: each step from the card's `## Verification
  Plan` (or the synthesized ad-hoc checklist, if the card predates it) with
  ✓/✗ and the actual output/value observed — not just a restated pass/fail.
- Acceptance criteria checklist (each ✓/✗, cross-referenced to the
  Verification Plan step(s) that proved it).
- Any uncertainties or items the agents flagged.

**Ask via `AskUserQuestion`:**

1. **Verdict**: `Approve — mark done` | `Let me test first` | `Needs changes: <feedback>`
2. If "Let me test first": offer to start the app (`npm run dev` for Vite) and
   give URL. Wait for user to return, then re-ask.
3. If "Needs changes": capture exact feedback.

### Phase 8 — Send-Back Loop (on rejection, max 2 cycles)

On rejection:
1. Analyze feedback + your own findings → create focused fix prompts for the
   relevant specialist agent(s).
2. Dispatch to **same agent(s)** (they have working-tree context):
   ```bash
   herdr agent prompt <agent_name> "<feedback + specific fix request>" --wait --timeout 300000
   ```
3. Re-run Phase 5 (verify) + Phase 6 (review) + Phase 7 (ask again).
4. Cap at 2 cycles. On 3rd failure, stop and present full history to user for
   decision.

### Phase 9 — On Approval: Mark Done & Close Agents

**Only on explicit "Approve — mark done":**
```bash
BACKLOG_PROJECT=<resolved> bash ~/.claude/skills/scripts/backlog done <UUID>
```

**Close every specialist agent's tab spawned in Phase 4** — since Phase 4
always opens a brand-new tab per agent (never a split off the orchestrator's
tab, never reused), every tab tracked there is yours to close **in full**, not
just the pane inside it:
```bash
herdr tab close <tab_id>
```
- Do NOT close while card is still in `review` — only after `done`.

### Phase 10 — Final Report

Report to user:
- Task UUID, title, board (Guiñotazo / Universiteit Utrecht), final state (`done`).
- Which agents participated (kind/model/agent-persona, effort level if
  Claude/Grok, fresh vs reused).
- Summary of changes per agent.
- Verification evidence (test count, lint, build, acceptance criteria).
- Human verdict + result (approved + done, or rejected + retries).
- Board URL: `http://127.0.0.1:8787/`.
- Optional: `herdr notification show "Worker" --body "Task <UUID> done"`.

---

## Usage as a Command

When invoked as `/worker`:
1. If task number provided as arg, use it; else ask (Phase 1).
2. Execute the full workflow above.
3. Interactive at Phase 1 (task pick), Phase 7 (verdict), and any send-back.

---

## Notes for the Agent Running This Skill

- **Never guess file paths** — read them first (`read`/`grep`/`glob`).
- **Never assume tooling** — detect and run verification command live (Phase 1).
- **Decomposition is key** — match sub-tasks to agent strengths; don't send
  everything to one agent.
- **Prompts must be self-contained** — specialist agents have zero shared
  context.
- **Every specialist agent gets its own brand-new tab, never a pane split off
  the orchestrator, never reused** — track every tab/agent from Phase 4 and
  close every tab in full (`herdr tab close`), only after `done`.
- **Two gates**: mechanical verification (Phase 5) + human approval (Phase 7).
- **Max 2 send-back cycles** — then escalate to user.
- **Anchor everything** — every claim about code must have `file:line` from your
  Phase 1/3 scans.
- **Engine purity is non-negotiable** — verify `src/engine` untouched (additive
  exports only via `src/engine/index.ts`).