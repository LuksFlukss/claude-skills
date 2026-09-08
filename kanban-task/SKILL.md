---
name: kanban-task
description: Deep discovery → parallel multi-agent solution proposals via Herdr → reconciled pitch (what was explored, what we'll do, why) → approval → push to kanban board. Use when you have a specific task and want multiple independent agents to find and explain the best solution before committing to a card.
---

# Kanban Task — Deep Discovery + Parallel Solution Proposals + Pitch + Board Push

Turn a raw idea/ticket/feature request into a **fully vetted, approved, and
pushed** Taskwarrior backlog card. This skill combines deep repo understanding,
independent parallel solution proposals from multiple Herdr agents, a
reconciled pitch that shows the user what was explored and why the chosen
approach wins, user approval gates, and finally pushing to the kanban board.

**The cardinal rule: clarify → dispatch parallel agents to independently
propose solutions → reconcile → pitch (what was explored, what we'll do, why
it wins) → draft → approve → push.** Never run `scripts/backlog add` until
the user has explicitly approved the final card after the pitch.

**Hard requirement — every agent this skill spins up, anywhere in the
workflow, is dispatched via the Herdr CLI using a kind/model picked from the
[Agent/Model Routing](#agentmodel-routing-strength--cost-based) table.** This
applies to Phase 1's repo-scan helpers just as much as Phase 3's
solution-proposal pair — there is no step in this skill where reaching for
the generic `Agent` tool (Claude Code subagents) instead of `herdr agent
start` is correct. Pick the agent from the table per the sub-task's type, and
— per the existing rule below — always tell the user which one you're
starting and why before you start it.

**Hard requirement — spun-up agents never share the orchestrator's pane or
tab.** An agent this skill starts must never run as a `herdr pane split` off
the orchestrator's own tab — splitting shares screen space with whatever else
is in that tab, and gets cramped/unreadable as agents pile in. Every agent
started by this skill — Phase 1's helpers included, not just Phase 3's
solution-proposal pair — gets its own brand-new tab (`herdr tab create`),
never a split pane, no exceptions. Once its work is done and it's no longer
needed, close its **entire tab** (`herdr tab close <tab_id>`), not just its
pane. See Phase 1, Phase 3.2, and Phase 9 for the exact commands.

---

## Before You Begin — Context Hygiene

This skill is long and multi-phase, and Phase 3 can spin up multiple Herdr
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
`kanban-task-eurocontrol`, not this one. That environment only has `claude`
installed as an agent kind (no OpenCode/Grok/Antigravity), and is a single
hardcoded board with no repo routing, so the dedicated skill drops all of
that machinery instead of trying to degrade gracefully here.

## Prerequisites

- Herdr CLI installed and `HERDR_ENV=1` in the environment (required for
  spawning solution-proposal agents).
- `~/.claude/skills/scripts/backlog` wrapper (project-agnostic; scopes every card via
  `BACKLOG_PROJECT`, see Board Routing below).
- Taskwarrior + taskwarrior-kanban backend running (`~/.task` store, shared
  across both boards — they're lanes on the same server, distinguished by
  Taskwarrior's `project:` field).

---

## Board Routing (Two Boards)

This skill's shared taskwarrior-kanban server currently hosts **two** boards
(lanes), selected by exporting `BACKLOG_PROJECT` before calling
`~/.claude/skills/scripts/backlog`:

| Repo | `BACKLOG_PROJECT` | Board name |
|------|-------------------|------------|
| `guiñote` (this repo) | `guiñotazo` | Guiñotazo |
| Any repo named `tf-*` (e.g. `tf-foo`, `tf-networking`) | `Universiteit Utrecht` | Universiteit Utrecht |

**Determine the target repo in Phase 1** (basename of the repo root you're
scanning, or ask the user if it's ambiguous which repo the card is for — e.g.
the ask spans multiple repos). Match it against the table above:

- Exact match `guiñote` → Guiñotazo board.
- Glob match `tf-*` → Universiteit Utrecht board.
- Anything else → ask the user which of the two boards the card belongs on
  (don't silently default to Guiñotazo just because it's the original board).

**Confirm the resolved board with the user immediately after Phase 1** (a
one-line "this'll be a Universiteit Utrecht card, correct?" is enough) —
**don't wait until Phase 5 to surface it.** Phase 3 can spin up paid Claude/
Grok agents; discovering the board was wrong only at the pitch means that
spend already happened against the wrong repo's conventions. Carry the resolved
`BACKLOG_PROJECT` value forward from there — it's needed at Phase 8 push time,
and still worth restating at Phase 5 for a final sanity check, just not as
the first time the user hears it.

**Universiteit Utrecht cards require a `repo:` field.** That board fans out
across every `tf-*` repo in the home directory, so a card with no repo
recorded on it is unusable by `worker` later — it has no way to know which
checkout to apply changes in. Resolve the absolute repo path in Phase 1/2
(the repo root you scanned, or ask if it's ambiguous) and carry it forward the
same way as `BACKLOG_PROJECT`; Phase 6's template and Phase 8's push both need
it. This does not apply to Guiñotazo (single-repo board).

---

## Agent/Model Routing (Strength & Cost Based)

Herdr has no built-in cost-aware scheduler — routing is a manual lookup this
skill performs using the task type classified in Phase 2. **See
[`../shared/agent-routing.md`](../shared/agent-routing.md) for the full
routing table, OpenCode invocation shape, effort selection, and
billing-failure fallback handling — read it before dispatching any agent in
this skill.** Use the table there to pick agents for **both** Phase 3
(solution proposals) and Phase 7 (build assignment) instead of defaulting to
the same pair every time.

**Classification (do this in Phase 2, right after clarifying the ask):** tag
the clarified task with one primary task type from the shared table's left
column (e.g. "multi-file implementation", "architecture", "focused fix",
"doc-heavy"). Carry this tag forward — it drives both the solution-proposal
pair (Phase 3) and the recommended builder (Phase 7).

**Solution-proposal pair (Phase 3) rule:** pick the best-fit agent for the
classified task type as one proposer, and a **different kind** with a **lower
or equal cost tier** as the second — diversity of kind matters more than raw
strength for surfacing genuinely different solutions, not just two takes on
the same one. Prefer a Free-tier second slot (MiMo) unless the task's risk
profile (engine purity, architecture) justifies spending a Claude slot. If the
best-fit agent and the natural second pick would both be `opencode`-kind (e.g.
Big Pickle + MiMo — the only two OpenCode-kind agents left in rotation),
consider swapping the second for **Grok** instead — a genuinely different
model family is more likely to propose a real alternative than an
opencode-vs-opencode pair. Never pick Claude for both proposal slots — it
wastes the parallel-exploration benefit and the cost budget for no extra
signal.

### Phase 0 — Ensure Kanban Board is Up

Before anything else, verify the board is reachable at `http://127.0.0.1:8787/`.
If not, run the `setup-kanban-board` skill (or equivalent steps) to start it.
The board is where the final card will be visible.

### Phase 1 — Read & Understand the Repo (Delegated Deep Scan)

**Do this before asking any clarifying questions.**

**Hard requirement — don't run this scan solo, and don't reach for the
generic `Agent` tool to delegate it.** Just like Phase 3 doesn't design the
solution alone, Phase 1 doesn't read the repo alone: spin up **helper agents
via Herdr**, picked from the [Agent/Model Routing](#agentmodel-routing-strength--cost-based)
table like every other agent this skill starts, to gather the picture in
parallel, and act as the **orchestrator** — dispatch, wait, reconcile.
Reaching for `grep`/`read`/`git log` yourself here, sub-task by sub-task,
defeats the point; only the one board-resolution check below is cheap enough
to do yourself.

0. **Resolve the board yourself first** — determine the repo name and
   resolve it to a `BACKLOG_PROJECT` per [Board Routing](#board-routing-two-boards).
   This is a one-line basename/glob check, not worth delegating, and every
   helper agent's backlog-lookup prompt below needs the resolved value before
   it can be dispatched.

1. **Dispatch helper agents in parallel via Herdr** — these are read-only
   research briefs, so per the Routing table's "Long doc/spec reading, quick
   lint/summarization" row, default each to **Big Pickle** (`opencode`
   → `-m opencode/big-pickle --agent plan`, low cost) unless a
   given scan clearly calls for deeper trade-off judgment, in which case use
   **MiMo V2.5** (`-m opencode/mimo-v2.5-free --agent plan`) instead — both
   forced to the read-only `plan` persona, since Phase 1 is discovery, not
   implementation. **Tell the user which agent you're starting for each scan,
   and why, before starting it** (same rule as everywhere else in this
   skill). Split the scan by concern, e.g.:
   - **Structure & conventions**: key directories/modules, how the repo is
     organized, languages/tools, and existing patterns for the kind of
     change likely being asked for (naming, module boundaries, state
     management, test patterns, CI checks). Have it also check for and read
     `AGENTS.md` for engine purity rules — e.g. "Keep the rules engine pure:
     bots/UI code must never mutate `src/engine` logic; engine changes are
     additive exports only (`src/engine/index.ts`)" — if that file exists.
   - **Tooling & live verification**: detect the project's tooling signals
     (`package.json`, `Makefile`, etc.), identify the exact verification
     command, and **run it once to capture live numbers** (test count, lint
     status) — never trust docs or memory for this.
   - **Recent direction & existing backlog**: `git log --oneline -20` for
     active work trajectory, plus `BACKLOG_PROJECT=<resolved project> bash
     ~/.claude/skills/scripts/backlog next` and `... board` to see current tasks,
     agents in use, priorities, and avoid duplication.

   Each scan gets its own brand-new tab, same as every other agent in this
   skill — never a split pane, never reused:
   ```bash
   TAB_JSON=$(herdr tab create --cwd "$PWD" --no-focus)
   TAB_ID=$(echo "$TAB_JSON" | jq -r '.result.tab.tab_id')
   PANE_ID=$(echo "$TAB_JSON" | jq -r '.result.root_pane.pane_id')
   herdr agent start scan-structure --kind opencode --pane "$PANE_ID" -- -m opencode/big-pickle --agent plan
   herdr agent prompt scan-structure "<self-contained brief>" --wait --timeout 300000
   ```
   Brief each agent like a colleague with zero context: give it the repo root,
   the resolved `BACKLOG_PROJECT`, and exactly what to report back —
   file:line-anchored facts, not vague summaries. These are research/read-only
   briefs, not implementation work, so say so explicitly in each prompt (same
   spirit as Phase 3.1's "DO NOT MODIFY ANY FILES" line, even though `plan`
   already blocks writes at the tool level).

   Track each helper's `tab_id`/`pane_id`/name the same way Phase 3 does —
   Phase 9 closes these tabs too, not just the Phase 3 solution-proposal
   pair's.

2. **Reconcile**: once every agent reports back (`herdr agent read <name>
   --source recent-unwrapped --lines 300`), merge their findings into one
   internal picture yourself. Resolve any contradiction directly (e.g. two
   agents disagreeing on the live test count — re-run the verification
   command yourself to settle it) rather than picking one report at random.

**Output**: Keep this internal. Use it to anchor every claim in the card to real
file:line references.

### Phase 2 — Clarify the Ask

Ask the user for the task. Accept free text, a pasted ticket, or rough notes.
Pull out missing essentials:

- What is the **outcome** (what should work/change when done)?
- Where does it live (which repo/folder)? Default: current repo — if
  `kanban-task` is being run from inside the target repo (the common case),
  this is just the cwd you already scanned in Phase 1; don't interrogate the
  user for something you can already see. **Only escalate to an explicit
  question when it's genuinely ambiguous** — the ask spans multiple repos, or
  you're authoring this card from a *different* repo than the one it targets.
  **If the card is going on the Universiteit Utrecht board and it's ambiguous
  which `tf-*` checkout is meant**, this is not optional to leave vague — pin
  down the exact absolute repo path before drafting; don't leave it as a
  bracketed placeholder like other open questions can be.
- Anything **out of scope** that should NOT be touched?
- Fresh feature, fix, refactor, or subtask of existing card?
- Any hard constraints (deadlines, blocking other work)?

**Task type classification** — tag the clarified ask with one primary type
from the [Agent/Model Routing](#agentmodel-routing-strength--cost-based) table
(e.g. "architecture", "multi-file implementation", "focused fix",
"doc-heavy"). This drives agent selection in Phase 3 and Phase 7.

**If, after pulling out the essentials above, the ask itself is still
genuinely ambiguous** (competing interpretations of the outcome, unclear
scope, or — as above — an unresolved `tf-*` repo target), ask **one**
clarifying round before moving on; don't manufacture disagreement or force a
back-and-forth when the ask is already clear. This is deliberately *not* a
multi-round design-sketching loop — Phase 3 dispatches real agents to explore
the solution space, and Phase 5's pitch is where the user reacts to actual
competing proposals grounded in the repo, which is a better moment to redirect
than an early hand-wavy sketch. Don't spend cycles re-deriving what Phase 3
will do better.

### Phase 3 — Parallel Solution Proposals (Herdr CLI)

**If `HERDR_ENV=1` is available**, spin up **two independent Herdr agents**
to each **independently propose a real solution** to the clarified ask — not
to critique a plan you already picked. Give both agents the same
self-contained brief and let them converge (or diverge) on their own; you
reconcile afterward in Phase 4.

#### 3.1 Prepare the proposal brief

Create a concise, self-contained prompt, shared verbatim by both agents:

```
REPO: <repo root path>
TASK: <user's ask, clarified in Phase 2>
CONSTRAINTS: <AGENTS.md rules, purity rules, verification commands>
EXISTING PATTERNS: <key patterns from Phase 1 to follow>
VERIFICATION COMMAND: <exact command to run, e.g. npm run lint && npm run test && npm run build>

Propose a concrete implementation approach for this task. Cover:
- What changes at a high level (files/modules touched, new resources/
  functions, anything removed) — anchor to real file:line where possible.
- Why this approach over the alternatives you considered — name at least one
  real alternative you weighed and say why you didn't pick it.
- How it fits existing conventions (naming, module boundaries, patterns above)
  rather than inventing new ones.
- What it deliberately does NOT handle (explicit non-goals).
- Risky areas — timing/order dependencies, purity boundaries, test
  implications.

DO NOT MODIFY ANY FILES. This is a design proposal only — describe the
approach and report back. Do not implement, fix, or refactor anything, even
if it looks trivial.
```

**Always include that "DO NOT MODIFY ANY FILES" line verbatim** — it's the
only thing standing between "proposal-only" and an agent that just goes ahead
and implements its own idea, especially once 3.2 launches it with a
full-tool-access persona for other reasons (see below).

#### 3.2 Choose effort (Claude/Grok only), then dispatch two agents via Herdr

Look up the Phase 2 task type in the **Agent/Model Routing** table to pick the
solution-proposal pair: the best-fit agent for that type, plus a
different-kind agent at equal-or-lower cost tier as the second proposer (see
"Solution-proposal pair rule" above). Do not default to the same claude+
opencode pair every time.

**Persona override for this phase — read the table's Herdr Kind for the
*model*, never for the persona:** the table's `--agent build` entries
(Big Pickle) are written for when that agent is later assigned to
actually *build* the card (Phase 7's recommendation, or the `worker` skill
picking it up afterward) — **not** for this proposal step. Every
OpenCode-kind agent launched in Phase 3, regardless of which row it came
from, **must use `--agent plan`** (read-only), never `build` — proposal-only
means the agent can't write files even if it wanted to, not just that the
prompt asks it not to. MiMo already defaults to `plan`; override Big
Pickle to `plan` here even though its table row says `build`.
Claude and Grok have no read-only persona concept — for those, the 3.1
prompt's "DO NOT MODIFY ANY FILES" line is the only guardrail, so never omit
it.

**If either slot landed on Claude or Grok**, first run the Effort Selection
step from the routing table above via `AskUserQuestion` — propose a
recommended effort per Claude/Grok agent, get the user's confirmation, *then*
start the agents with the chosen `--effort`. OpenCode-kind slots skip this and
go straight to launch with `-m`/`--agent`.

**Never split a pane off the orchestrator's tab, and never reuse an idle
pane — every proposal agent gets a brand-new tab of its own.** A pane split
shares screen space with the orchestrator's own view (and any other agent
already in that tab) — it gets cramped and cluttered as agents pile in. An
idle-looking pane may also belong to the user's own session or another agent;
reusing it risks clobbering work-in-progress or mixing two agents' context
together. `herdr tab create` per agent, every time, no exceptions. Start each
agent in its own new tab, then send the Phase 3.1 prompt:

```bash
# Example: task classified as "multi-file implementation" ->
# proposer A = Big Pickle (best-fit, low cost). The natural second pick (MiMo)
# would also be opencode-kind, so per the swap rule, proposer B = Grok instead
# (different model family, needs an effort choice per the Effort Selection step).
# Both OpenCode-kind agents are forced to --agent plan here (read-only) — this
# is a proposal, not the build.
TAB_A_JSON=$(herdr tab create --cwd "$PWD" --no-focus)          # -> tab A
TAB_A_ID=$(echo "$TAB_A_JSON" | jq -r '.result.tab.tab_id')
PANE_A_ID=$(echo "$TAB_A_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start propose-a --kind opencode --pane "$PANE_A_ID" -- -m opencode/big-pickle --agent plan

TAB_B_JSON=$(herdr tab create --cwd "$PWD" --no-focus)          # -> tab B
TAB_B_ID=$(echo "$TAB_B_JSON" | jq -r '.result.tab.tab_id')
PANE_B_ID=$(echo "$TAB_B_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start propose-b --kind grok --pane "$PANE_B_ID" -- --effort medium

# Example: task classified as "architecture" with engine-purity risk ->
# proposer A = Claude (paid, correctness-critical) at user-confirmed effort,
# proposer B = MiMo (free, different kind, plan persona)
herdr agent start propose-a --kind claude --pane "$PANE_A_ID" -- --effort high
herdr agent start propose-b --kind opencode --pane "$PANE_B_ID" -- -m opencode/mimo-v2.5-free --agent plan

herdr agent prompt propose-a "$(cat /tmp/proposal_prompt.md)" --wait --timeout 600000
herdr agent prompt propose-b "$(cat /tmp/proposal_prompt.md)" --wait --timeout 600000
```

Track each proposal agent's `tab_id` alongside its `pane_id`/name — Phase 9
needs the `tab_id` to close it out.

Wait for both to complete (poll `herdr agent get <name>` for `agent_status:
idle` if `--wait` times out on a long-running one — this is normal for a
substantial proposal, not a failure). Collect their reports (`herdr agent
read <name> --source recent-unwrapped --lines 3000 --format text`; if the
pane's own scrollback is truncated, ask the agent to re-state its proposal in
a shorter follow-up prompt rather than relying on a long single response —
the plan persona can't write a file for you to read back).

Note the tab/pane/agent identifiers Herdr assigns to each of these two
proposal agents (`herdr agent list` / `herdr tab list`) — you'll need the
`tab_id`s in Phase 9 to close these agents out once the card is pushed.

**If an agent is out of credits / billing-failed**, follow
`../shared/agent-routing.md`'s "Billing-Failure / Rate-Limit Fallback"
section — same rule, this phase's proposal slot is just the thing being
re-dispatched. Re-dispatch to whichever the user picks before moving to
Phase 4.

**If no Herdr / HERDR_ENV=1**: design the solution yourself (same bullet list
as the 3.1 brief above) and skip to Phase 4 noting only one internal proposal
exists — flag this in the eventual card (user can request multi-agent
proposals later).

### Phase 4 — Reconcile & Synthesize

Read both proposals and produce **one** synthesized solution — this becomes
your internal design, grounded in what the two agents actually proposed
rather than invented from scratch:

- **Common ground** (both proposals agree) → lock in.
- **Divergences** (the two proposals disagree on a decision) → resolve using
  convention fit (Phase 1), risk, and simplicity — decide, don't average.
- **Better ideas either proposal surfaced that you hadn't considered** →
  adopt them.
- **Non-goals / risks** from either proposal that still apply → carry forward.

Write this up as the same shape of design doc the old solo Phase 3 produced
(what changes, why, how it fits conventions, non-goals, risky areas) — the
difference is every claim in it should trace back to one or both of the real
proposals, not to your own unassisted judgment.

**Optional escalation, not a default step:** if the two proposals genuinely
conflict in a way that's hard to resolve from convention/risk alone, or the
task is classified "architecture" with engine-purity or infra-state risk,
you may spin up **one** additional Herdr agent (routed per the table, same
`--agent plan` + "DO NOT MODIFY ANY FILES" rules as Phase 3) to sanity-check
the *synthesized* plan before pitching it. This is a narrow escape hatch, not
a replacement for the old critique-only phase — most tasks don't need it, and
defaulting to it every time reintroduces the cost/latency this restructuring
removed.

**Mandatory verify pass — narrower than the escalation above (fact-checking,
not re-design), but fires on a concrete trigger rather than a judgment call,
so it fires more often:** spin up **one** Herdr agent (same routing/
`--agent plan`/"DO NOT MODIFY ANY FILES" rules) to check the synthesized
plan's specific technical claims against the live repo — not to re-litigate
the whole design — whenever **either** holds:

- The task touches a template/shared-infra repo consumed by more than one
  other repo (real blast radius beyond this repo), or
- The synthesis depends on a specific technical claim (a function/API/
  syntax/tool behavior) that only one of the two Phase 3 proposals actually
  verified — not just asserted — and adopting it into the synthesis means
  trusting that single, unverified source.

Brief this agent narrowly — name the exact claims to check (e.g. "confirm
the Azure DevOps `iif()` template function collapses this exact pattern the
way the plan assumes" / "confirm these file:line anchors still match current
HEAD" / "confirm this parameter name doesn't collide with anything the 3
consumer repos already pass") rather than handing it the whole plan to
re-review. If it disproves something the plan depends on, that's a
correction to fold back into the synthesis before pitching — not new scope,
and not grounds to restart Phase 3.

**Do not present this yet.** It's pitched, with its provenance, in Phase 5.

### Phase 5 — Pitch: What We Explored, and Why This Wins

Present the synthesized solution to the user as a clear, conversational
pitch — not a formal doc yet, and not just "here's the plan" — the point is
to make the exploration visible, not just the conclusion. Include:

**## What I Explored**

A short table, one row per Phase 3 proposal (plus the optional Phase 4
sanity-check agent if one ran):

| # | Approach | Proposed by | Key trade-off |
|---|----------|-------------|----------------|
| A | <short name> | <agent> | <pro vs. con> |
| B | <short name> | <agent> | <pro vs. con> |

**## What I'll Do (and Why It Wins)**

2-4 sentences naming which proposal(s) the synthesis draws from and why it
beats the alternative(s) — reference the actual trade-offs from the table
above, not hypothetical ones. If Phase 3 ran with only one internal proposal
(no Herdr), say so plainly instead of fabricating a comparison.

**## Verification Plan, Risks, Non-Goals**

Same as before: exact commands, expected test count, open questions, explicit
out-of-scope items.

**## Rollout & Blast Radius** (mandatory whenever the card touches a
template/shared-infra repo consumed by more than one other repo, or
otherwise reaches beyond this repo — omit entirely for a self-contained
single-repo fix, don't pad it in where it doesn't apply):

- **Affected**: the actual list of consuming repos/environments — not
  "downstream systems" as a vague placeholder.
- **Rollout**: how the change reaches them safely (a version tag, an
  opt-in/default-off flag, a staged rollout) — never "it just lands on
  everyone's next run" for a behavior change with real consequences.
- **Rollback**: what undoing this looks like if a consumer breaks after
  adopting it.

Ask the user to react: **Approve as-is**, **Show me the raw agent reports**
(pull the full Phase 3 outputs before deciding), **Request changes**, or
**Pick the other explored approach instead**. Iterate here — refine the
synthesis based on feedback — until the user explicitly approves.

**Use `AskUserQuestion` with explicit options:**
- `Approve — proceed to card drafting`
- `Show me the raw agent reports`
- `Request changes` (user describes adjustments)
- `Pick approach [B] instead` (only offer if a real rejected alternative exists)

### Phase 6 — Draft the Final Card (Using the Template)

Once the user approves the solution, write the complete card using the
template at [`../shared/card-template.md`](../shared/card-template.md) — read
it now if you haven't already. Fill every applicable section; omit ones that
genuinely don't apply. Keep it dense but readable. **Every file reference
must have a line anchor.**

**The `## Verification Plan` section is mandatory, not optional** — never
substitute it with just "run the test suite" when the task touches anything a
generic test/lint/build command can't observe (infra state, external
resources, DB rows, migrations, third-party config). Write it so a builder
with zero conversation context — including the `worker` skill executing this
card later — can mechanically confirm each Acceptance Criterion is actually
met, not just that the build didn't break.

**The `## Rollout & Blast Radius` section is mandatory whenever the card
touches a template/shared-infra repo consumed by more than one other repo**
(carry it over verbatim from Phase 5's pitch — this is not new work, just
transcription) — omit it entirely for a self-contained single-repo change.

The shared template's `<Universiteit Utrecht cards only>` line references
"Phase 8" for the push step and the `worker` skill for repo resolution —
that's this skill's Phase 8, unchanged.

### Phase 7 — Final Approval Gate (Card Push)

Present the **complete drafted card** in the chat. **Do NOT push yet.**

**If this is a Universiteit Utrecht card and Phase 6's `Repo:` line is still a
placeholder or missing**, resolve that before asking anything else — it's a
hard requirement, not a nice-to-have; `bash ~/.claude/skills/scripts/backlog add`
will refuse the push without it (Phase 8).

Use `AskUserQuestion` (single call, up to 4 questions) to get:

1. **Approve card?** — Options: `Approve — push it as drafted` (recommended),
   `Request changes` (user describes adjustments; loop back to Phase 6).
2. **Which agent should build it?** — Compute a **recommendation** from the
   Agent/Model Routing table using the Phase 2 task-type classification (e.g.
   "multi-file implementation" → Big Pickle recommended). Present that as the
   first option, labeled `<Agent name> — recommended (<task type>, <cost tier>)`.
   Then look at recent cards (`bash scripts/backlog next` / `bash scripts/backlog
   board`) for other agent names actually in use and offer up to two of those as
   alternatives, plus `Unassigned — leave agent blank`. The tool's "Other" choice
   covers any name not listed. Routing recommends; it does not decide — the user
   still picks.
3. **What priority?** — Options: `High` (blocking/urgent), `Medium` (normal
   backlog work), `Low` (nice-to-have). Don't default silently — if the card
   implied urgency, note it in the description but still let the user pick.

**Only on "Approve" → proceed to Phase 8.**

### Phase 8 — Push to the Board

Export the `BACKLOG_PROJECT` resolved in Board Routing, then push:

```bash
export BACKLOG_PROJECT="<guiñotazo|Universiteit Utrecht>"
bash ~/.claude/skills/scripts/backlog add "<full card text>" --priority <H|M|L> [--agent <model>] [--repo <absolute path>]
```

- `--priority`: H/M/L from user's choice.
- `--agent`: the name chosen; **omit entirely** if "Unassigned" was picked.
- `--repo`: **required for Universiteit Utrecht**, the absolute repo path
  resolved in Phase 2/6. The wrapper refuses the push without it on that
  board (it has no such requirement on Guiñotazo — omit there).

Confirm back to the user:
- New card ID, state `todo`, assigned builder (`agent:`), and **which board**
  it landed on (Guiñotazo vs. Universiteit Utrecht).
- Board URL: `http://127.0.0.1:8787/` (both boards live on the same server;
  the `project:` field distinguishes lanes).
- Commands to pick it up (same `BACKLOG_PROJECT` exported): `bash
  ~/.claude/skills/scripts/backlog claim`, then `review`/`done`.

### Phase 9 — Clean Up the Herdr Agents

Once the card push is confirmed, close out **every** Herdr agent this skill
spun up — Phase 1's repo-scan helpers, Phase 3's solution-proposal pair, and
Phase 4's optional sanity-check agent if one ran — they've served their
purpose and shouldn't linger:

```bash
herdr agent list         # confirm every tab/pane/agent from Phase 1, 3, and 4 (by id/name)
herdr tab close <tab_id>   # once per agent's tab spun up in Phase 1, 3, or 4
```

Do this for **every** agent started in Phase 1, 3, or 4 (skip whichever
phase's agents don't apply — e.g. Phase 3 was skipped because `HERDR_ENV=1`
was unavailable). Confirm to the user that all of them have been closed.

---

## Quality Bar — The Card Must Enable Zero-Context Execution

An agent should be able to hand this card to a fresh session and see exactly:

- **What** to build (Goals + Context)
- **Where** (Current State with real file:line anchors)
- **How** (Implementation Guidance)
- **What to avoid** (Constraints / Non-Goals + engine purity rules)
- **When it's finished** (Acceptance Criteria with confirmed tooling + live test counts)
- **How to confirm it's finished** (the per-criterion `Verification Plan` — concrete
  checks a fresh builder/session can run and compare against a stated expected
  result, not just "the test suite passed")
- **Solution pedigree** (which agents proposed what, and why the chosen
  approach won — or a note that only one internal proposal existed)
- **Blast radius, when it exists** (`## Rollout & Blast Radius` — who's
  affected, how the change reaches them safely, how to roll it back; omitted
  only when the change is genuinely self-contained to this one repo)

And the user **always** signs off at two gates: solution pitch (Phase 5) and
final card (Phase 7).

---

## Usage as a command

When invoked as `/kanban_task`, execute the full workflow above. The skill is
interactive — it will ask questions at Phase 2 (essentials, plus one
clarifying round only if genuinely ambiguous), Phase 5 (pitch approval), and
Phase 7 (final card approval) via `AskUserQuestion` or open conversation. Do
not skip any phase.

---

## Notes for the Agent Running This Skill

- **Never guess file paths** — read them first (use `read`/`grep`/`glob`).
- **Never assume tooling** — detect and confirm with the user (Phase 1).
- **Every agent this skill starts — Phase 1's repo-scan helpers included, not
  just Phase 3's solution-proposal pair — is picked from the Agent/Model
  Routing table and started via `herdr agent start`.** The generic `Agent`
  tool is never the right call anywhere in this workflow.
- **Never invent a solution alone when Herdr is available** — Phase 3 dispatches
  two independent agents to *propose* real solutions before you write anything
  down; Phase 4's synthesis must trace every claim back to what they actually
  proposed, not to your own unassisted judgment. Only design solo (skip to
  Phase 4 with one internal proposal) when `HERDR_ENV=1` is unavailable.
- **Never push without approval** — two explicit gates (Phase 5's pitch, Phase
  7's final card).
- **Solution-proposal agents never write files** — Phase 3 is proposal-only:
  OpenCode-kind proposers always launch with `--agent plan` regardless of what
  the Routing table's row says (that persona is for the eventual builder, not
  the proposer), and the 3.1 prompt always includes the "DO NOT MODIFY ANY
  FILES" line for every agent kind.
- **The parallel solution-proposal step is mandatory when Herdr is available**
  — if `HERDR_ENV=1` is set but Herdr fails, surface the error and ask the
  user whether to proceed with a single internal proposal or retry.
- **Fact-check the synthesis before pitching it, when the trigger fires** —
  Phase 4's mandatory verify pass (blast radius beyond this repo, or a
  technical claim only one proposal verified) is narrower than the optional
  full-plan sanity-check above it, but not skippable when its trigger holds.
- **Name the blast radius, don't let it hide in prose** — a card touching a
  shared/template repo with more than one consumer gets its own
  `## Rollout & Blast Radius` section (Phase 5 pitch and Phase 6 card); a
  single-repo fix omits it rather than padding one in.
- **Keep changes uncommitted** — the card's acceptance criteria includes this.
- **Anchor everything** — every claim about current code must have a `file:line`
  reference from your Phase 1 scan.
- **Close Herdr agents after the push** — Phase 3 spins up the solution-proposal
  pair (and Phase 4 may spin up one optional sanity-check agent); once Phase 8
  confirms the card is pushed, close those agents/panes (Phase 9) so they
  don't linger unused.
- **Every agent — Phase 1 helpers and Phase 3 proposers alike — gets its own
  brand-new tab, never a pane split off the orchestrator, never reused** — no
  exceptions; this is also why every one of them gets closed **in full**
  (`herdr tab close`) in Phase 9 (nothing there was ever the user's own pane
  or tab).