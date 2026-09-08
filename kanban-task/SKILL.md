---
name: kanban-task
description: Deep discovery → solution design → multi-agent Herdr verification → iterative pitch → approval → push to kanban board. Use when you have a specific task and want expert verification before committing to a card.
---

# Kanban Task — Deep Discovery + Solution Pitch + Herdr Verification + Board Push

Turn a raw idea/ticket/feature request into a **fully vetted, approved, and
pushed** Taskwarrior backlog card. This skill combines deep repo understanding,
iterative solution design, multi-agent best-practice verification via Herdr CLI,
user approval gates, and finally pushing to the kanban board.

**The cardinal rule: clarify → co-design with the user → draft → verify → pitch
→ approve → push.** Never run `scripts/backlog add` until the user has
explicitly approved the final card after all verification rounds.

**Hard requirement — every agent this skill spins up, anywhere in the
workflow, is dispatched via the Herdr CLI using a kind/model picked from the
[Agent/Model Routing](#agentmodel-routing-strength--cost-based) table.** This
applies to Phase 1's repo-scan helpers just as much as Phase 4's verification
pair — there is no step in this skill where reaching for the generic `Agent`
tool (Claude Code subagents) instead of `herdr agent start` is correct. Pick
the agent from the table per the sub-task's type, and — per the existing rule
below — always tell the user which one you're starting and why before you
start it.

**Hard requirement — verification agents never share the orchestrator's pane
or tab.** A verification agent must never run as a `herdr pane split` off the
orchestrator's own tab — splitting shares screen space with whatever else is
in that tab, and gets cramped/unreadable as agents pile in. Every agent
started by this skill — Phase 1's helpers included, not just Phase 4's
verification pair — gets its own brand-new tab (`herdr tab create`), never a
split pane, no exceptions. Once its work is done and it's no longer needed,
close its **entire tab** (`herdr tab close <tab_id>`), not just its pane. See
Phase 1, Phase 4.2, and Phase 9 for the exact commands.

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
`kanban-task-eurocontrol`, not this one. That environment only has `claude`
installed as an agent kind (no OpenCode/Grok/Antigravity), and is a single
hardcoded board with no repo routing, so the dedicated skill drops all of
that machinery instead of trying to degrade gracefully here.

## Prerequisites

- Herdr CLI installed and `HERDR_ENV=1` in the environment (required for
  spawning verification agents).
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
**don't wait until Phase 5 to surface it.** Phase 4 spins up paid Claude/Grok
agents; discovering the board was wrong only at the pitch means that spend
already happened against the wrong repo's conventions. Carry the resolved
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
skill performs using the task type classified in Phase 3. Use this table to
pick agents for **both** Phase 4 (verification) and Phase 7 (build assignment)
instead of defaulting to the same pair every time.

| Task Type | Best-fit Agent | Herdr Kind / Model / OpenCode Agent | Cost Tier | Rationale |
|-----------|---------------|---------------------|-----------|-----------|
| Architecture / high-level design, trade-off analysis | MiMo V2.5 | `opencode` → `-m opencode/mimo-v2.5-free --agent plan` | Free | Strong reasoning/planning at no cost — use for design-heavy tasks. `--agent plan` is OpenCode's read-only design/reasoning persona, matching this task type. |
| Complex multi-file implementation, large refactors | Claude | `claude` (Sonnet/Opus) | $$ (paid) | Best correctness on architecture-sensitive or engine-purity-sensitive changes |
| Everyday multi-file coding, focused bug fixes, test writing | Big Pickle | `opencode` → `-m opencode/big-pickle --agent build` | $ (low) | Good default for routine implementation, cheaper than Claude. `--agent build` is OpenCode's full-tool-access implementation persona. |
| Long doc/spec reading, quick lint/summarization | Nemotron 3 Ultra | `opencode` → `-m opencode/nemotron-3-ultra-free --agent build` | Free | Fast and cheap for read-heavy or mechanical sub-tasks. (A faster/lighter alternative exists: `opencode/nemotron-3.5-lightning-free` — swap in if speed matters more than thoroughness for the specific sub-task.) |
| Alternative cross-check / diverse second opinion, broad-context reasoning | Grok | `grok` | $ (paid) | Different model family from the rest of the table — good diversity pick for a verification cross-check when both other slots would otherwise be opencode-based. |
| Fallback (used only when Grok, or any OpenCode-kind agent above — MiMo/Big Pickle/Nemotron — hits its usage/rate limit or runs out of tokens; never a first pick) | Google Antigravity | `agy` → `--model <name> --effort <level>` (run `agy models` first to confirm current model names — prefer a `-pro-` tier for cross-check work, e.g. `gemini-3.1-pro-high`) | Google-account quota | Different model family from every other row — substituted in only when the agent it's replacing is confirmed unavailable. |

**Whenever ANY agent from this table is dispatched — Grok included, but also
Claude/Big Pickle/MiMo/Nemotron/Antigravity — explicitly tell the user which
one (and briefly why) before starting it.** Never fold an agent selection
into a pane-split/start sequence silently.

**Classification (do this in Phase 3):** tag the drafted approach with one
primary task type from the left column (e.g. "multi-file implementation",
"architecture", "focused fix", "doc-heavy"). Carry this tag forward — it drives
both the verification pair (Phase 4) and the recommended builder (Phase 7).

**Cost discipline:** prefer the Free tier whenever the task type matches
(MiMo/Nemotron) rather than defaulting to Claude. Only route to Claude when
the task is genuinely complex multi-file work, touches `src/engine` purity
rules, or the classification is itself "architecture" at high risk. Never pick
Claude for both verification slots — it wastes the parallel-diagnosis benefit
and the cost budget for no extra signal.

**Verification pair (Phase 4) rule:** pick the best-fit agent for the
classified task type as one verifier, and a **different kind** with a **lower
or equal cost tier** as the cross-check — diversity of kind matters more than
raw strength for catching divergent issues. Prefer two Free-tier agents unless
the task's risk profile (engine purity, architecture) justifies spending a
Claude slot. If the best-fit agent and the natural cross-check would both be
`opencode`-kind (e.g. Big Pickle + Nemotron), consider swapping the cross-check
for **Grok** instead — a genuinely different model family catches divergent
issues an opencode-vs-opencode pair might both miss.

**OpenCode invocation shape:** always launch OpenCode-kind agents with both
`-m <provider/model>` (the model, full `provider/model` form — bare model
names are not guaranteed to resolve) **and** `--agent <name>` (OpenCode's own
agent persona, separate from the model choice). Only `build` and `plan` are
launchable top-level personas for our purposes (`explore`/`general` are
subagents OpenCode dispatches internally, not directly startable via this
flag) — use `build` for any sub-task that needs to read/write files or run
commands (implementation, testing, lint fixes), and `plan` for a sub-task that
is pure analysis/design/reasoning with no file changes expected. Via Herdr:
`herdr agent start <name> --kind opencode --pane <id> -- -m opencode/<model>
--agent <build|plan>`.

**Effort selection (Claude & Grok only — OpenCode has no effort concept):**
Before starting **any** Claude or Grok agent in this skill (Phase 4's
verification pair, whichever slots landed on Claude/Grok per the table above),
propose a recommended effort level based on the task's actual complexity —
don't just default to the tool's default effort. Then confirm with the user
via `AskUserQuestion` before spinning the agent up (one question per Claude/Grok
agent about to be started, batched into as few calls as fits the 4-question
limit). Present the recommended level first, labeled `<level> — recommended
(<one-line reason>)`, with 1-2 adjacent levels as alternatives.

- **Claude** accepts `--effort <level>`: `low`, `medium`, `high`, `xhigh`, `max`
  (confirmed via `claude --help`). Recommend `medium` for routine verification
  of a well-scoped design, `high` for genuinely complex/architecture-risk
  diagnosis, `xhigh`/`max` only for a design this skill's own Phase 3 flagged
  as unusually high-risk (e.g. security/trust-boundary or engine-purity-critical).
- **Grok** accepts `--reasoning-effort <level>` (alias `--effort`); OpenCode's
  CLI help does not enumerate exact accepted values, but `low`/`medium`/`high`
  are confirmed to work in practice (a pane's status line showed `Grok 4.6
  (high)` after using `high`). Recommend the same way as Claude's scale,
  substituting `high` where Claude would get `xhigh`/`max` (Grok has no higher
  tier than `high`).
- **Google Antigravity** (`agy` — Grok's designated fallback, Phase 4.2) also
  accepts `--effort <level>`: `low`, `medium`, `high` (confirmed via
  `agy --help`) — same three-tier scale as Grok, no `xhigh`/`max`. It also
  needs `--model <name>` (run `agy models` to list current options; prefer a
  `-pro-` tier for cross-check work). If Antigravity is being started as a
  mid-workflow replacement for a failed Grok, ask the effort question the same
  way, just labeled for Antigravity instead.
- Launch with the chosen level: `herdr agent start <name> --kind claude --pane
  <id> -- --effort <level>` / `herdr agent start <name> --kind grok --pane <id>
  -- --effort <level>` / `herdr agent start <name> --kind agy --pane <id> --
  --model <model-name> --effort <level>`.

### Phase 0 — Ensure Kanban Board is Up

Before anything else, verify the board is reachable at `http://127.0.0.1:8787/`.
If not, run the `setup-kanban-board` skill (or equivalent steps) to start it.
The board is where the final card will be visible.

### Phase 1 — Read & Understand the Repo (Delegated Deep Scan)

**Do this before asking any clarifying questions.**

**Hard requirement — don't run this scan solo, and don't reach for the
generic `Agent` tool to delegate it.** Just like Phase 4 doesn't diagnose the
design alone, Phase 1 doesn't read the repo alone: spin up **helper agents
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
   lint/summarization" row, default each to **Nemotron 3 Ultra** (`opencode`
   → `-m opencode/nemotron-3-ultra-free --agent plan`, free tier) unless a
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
   herdr agent start scan-structure --kind opencode --pane "$PANE_ID" -- -m opencode/nemotron-3-ultra-free --agent plan
   herdr agent prompt scan-structure "<self-contained brief>" --wait --timeout 300000
   ```
   Brief each agent like a colleague with zero context: give it the repo root,
   the resolved `BACKLOG_PROJECT`, and exactly what to report back —
   file:line-anchored facts, not vague summaries. These are research/read-only
   briefs, not implementation work, so say so explicitly in each prompt (same
   spirit as Phase 4.1's "DO NOT MODIFY ANY FILES" line, even though `plan`
   already blocks writes at the tool level).

   Track each helper's `tab_id`/`pane_id`/name the same way Phase 4 does —
   Phase 9 closes these tabs too, not just the Phase 4 verification pair's.

2. **Reconcile**: once every agent reports back (`herdr agent read <name>
   --source recent-unwrapped --lines 300`), merge their findings into one
   internal picture yourself. Resolve any contradiction directly (e.g. two
   agents disagreeing on the live test count — re-run the verification
   command yourself to settle it) rather than picking one report at random.

**Output**: Keep this internal. Use it to anchor every claim in the card to real
file:line references.

### Phase 2 — Clarify the Ask, Then Co-Design the Idea With the User (2-4 Rounds)

This phase has two parts: first pull out the missing facts, then — before any
paid Herdr agent ever spins up — spend a few rounds actually thinking the idea
through *with* the user, not just at them.

#### 2.1 Clarify the essentials

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

#### 2.2 Co-design loop — challenge the idea before committing to it (~2-4 rounds)

Once you have enough to sketch an approach, **don't jump straight to a
locked-in design.** Think out loud with the user first — this is cheap (no
Herdr agents involved yet) and is where the biggest wins usually come from,
before Phase 3-4 sink real effort/cost into stress-testing a design that
turns out to be the wrong one.

Each round:

1. **Present a short idea sketch** (a few sentences, not the full Phase 3
   write-up): the approach you're leaning toward, anchored to real
   `file:line` where relevant.
2. **Name at least one real alternative or trade-off** you considered — don't
   just present one path as if it were the only option. If you don't see a
   plausible alternative, say so explicitly rather than skipping this.
3. **Actively ask the user to poke holes in it** — not a yes/no rubber stamp.
   Use `AskUserQuestion` or plain conversation with something like: "Is there
   anything you'd do differently here? A simpler way? Something I'm
   missing?" — genuinely invite the "actually, what about X" response, don't
   just wait for a rubber-stamp approval.
4. **Incorporate whatever comes back** and refine the sketch before the next
   round.

Repeat for **roughly 2-4 rounds** — fewer if the user converges quickly
("looks good, go with it" — take that at face value and stop, don't force
extra rounds for their own sake), more only if round 4 still surfaced a real
open question. The point is genuine convergence on a good idea, not hitting a
quota.

**Only once the user has explicitly signed off on the shape of the idea**
("yes let's go with this") do you move to Phase 3 and the rest of the
agent-heavy flow (internal detailed design → Herdr verification → pitch →
card → push). Phase 5's later approval gate is about the *fully verified,
drafted* solution — this loop is about not wasting Phase 4's paid agents on
an idea the user would have redirected anyway.

**If the idea is clear enough after 2.1 that there's really nothing to
challenge** (a small, unambiguous fix with one obvious approach), a single
round confirming "here's the approach, sound right?" is enough — don't
manufacture disagreement. Flag any remaining genuinely open questions as
bracketed placeholders `[...]` for confirmation at Phase 7, rather than
blocking on them here.

### Phase 3 — Design a Best-Practices Solution (Internal)

Using the repo understanding (Phase 1) and the clarified ask (Phase 2), design a
concrete implementation approach:

- **What changes** at a high level (files/modules touched, new
  resources/functions, anything removed).
- **Why this approach** over obvious alternatives — call out the trade-off.
- **How it fits existing conventions** (naming, module boundaries, patterns
  from Phase 1) rather than inventing new ones.
- **What it deliberately does NOT handle** (explicit non-goals).
- **Risky areas** — timing/order dependencies, purity boundaries, test
  implications.
- **Task type classification** — tag this design with one primary type from
  the [Agent/Model Routing](#agentmodel-routing-strength--cost-based) table
  (e.g. "architecture", "multi-file implementation", "focused fix",
  "doc-heavy"). This drives agent selection in Phase 4 and Phase 7.

**Do not present this yet.** This is your internal design that will be
stress-tested in Phase 4.

### Phase 4 — Multi-Agent Best-Practice Verification (Herdr CLI)

**If `HERDR_ENV=1` is available**, spin up **two independent Herdr agents**
(typically different kinds, e.g. a Claude session and an OpenCode session) to
investigate the proposed solution in parallel. This is a **diagnosis-only**
step — agents do NOT apply fixes.

#### 4.1 Prepare the verification prompt

Create a concise, self-contained prompt for the verification agents containing:

```
REPO: <repo root path>
TASK: <user's ask, clarified>
PROPOSED APPROACH: <your Phase 3 design, with file:line anchors>
CONSTRAINTS: <AGENTS.md rules, purity rules, verification commands>
EXISTING PATTERNS: <key patterns from Phase 1 to follow>
VERIFICATION COMMAND: <exact command to run, e.g. npm run lint && npm run test && npm run build>

DO NOT MODIFY ANY FILES. This is diagnosis only — critique the proposed
approach (soundness, edge cases, better alternatives) and report back. Do not
implement, fix, or refactor anything, even if the fix looks trivial.
```

**Always include that "DO NOT MODIFY ANY FILES" line verbatim** — it's the
only thing standing between "diagnosis-only" and an agent that just goes
ahead and implements the change, especially once 4.2 launches it with a
full-tool-access persona for other reasons (see below).

#### 4.2 Choose effort (Claude/Grok only), then dispatch two agents via Herdr

Look up the Phase 3 task type in the **Agent/Model Routing** table to pick the
verification pair: the best-fit agent for that type, plus a different-kind
agent at equal-or-lower cost tier as the cross-check (see "Verification pair
rule" above). Do not default to the same claude+opencode pair every time.

**Persona override for this phase — read the table's Herdr Kind for the
*model*, never for the persona:** the table's `--agent build` entries
(Big Pickle, Nemotron) are written for when that agent is later assigned to
actually *build* the card (Phase 7's recommendation, or the `worker` skill
picking it up afterward) — **not** for this verification step. Every
OpenCode-kind agent launched in Phase 4, regardless of which row it came
from, **must use `--agent plan`** (read-only), never `build` — diagnosis-only
means the agent can't write files even if it wanted to, not just that the
prompt asks it not to. MiMo already defaults to `plan`; override Big
Pickle/Nemotron to `plan` here even though their table row says `build`.
Claude and Grok have no read-only persona concept — for those, the 4.1
prompt's "DO NOT MODIFY ANY FILES" line is the only guardrail, so never omit
it.

**If either slot landed on Claude or Grok**, first run the Effort Selection
step from the routing table above via `AskUserQuestion` — propose a
recommended effort per Claude/Grok agent, get the user's confirmation, *then*
start the agents with the chosen `--effort`. OpenCode-kind slots skip this and
go straight to launch with `-m`/`--agent`.

**Never split a pane off the orchestrator's tab, and never reuse an idle
pane — every verification agent gets a brand-new tab of its own.** A pane
split shares screen space with the orchestrator's own view (and any other
agent already in that tab) — it gets cramped and cluttered as agents pile in.
An idle-looking pane may also belong to the user's own session or another
agent; reusing it risks clobbering work-in-progress or mixing two agents'
context together. `herdr tab create` per agent, every time, no exceptions.
Start each agent in its own new tab, then send the Phase 4.1 prompt:

```bash
# Example: task classified as "multi-file implementation" ->
# primary = Big Pickle (best-fit, low cost), cross-check = Nemotron (free, different kind)
# Both forced to --agent plan here (read-only) — this is diagnosis, not the build.
TAB_A_JSON=$(herdr tab create --cwd "$PWD" --no-focus)          # -> tab A
TAB_A_ID=$(echo "$TAB_A_JSON" | jq -r '.result.tab.tab_id')
PANE_A_ID=$(echo "$TAB_A_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start verify-a --kind opencode --pane "$PANE_A_ID" -- -m opencode/big-pickle --agent plan

TAB_B_JSON=$(herdr tab create --cwd "$PWD" --no-focus)          # -> tab B
TAB_B_ID=$(echo "$TAB_B_JSON" | jq -r '.result.tab.tab_id')
PANE_B_ID=$(echo "$TAB_B_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start verify-b --kind opencode --pane "$PANE_B_ID" -- -m opencode/nemotron-3-ultra-free --agent plan

# Example: task classified as "architecture" with engine-purity risk ->
# primary = Claude (paid, correctness-critical) at user-confirmed effort,
# cross-check = MiMo (free, different kind, plan persona)
herdr agent start verify-a --kind claude --pane "$PANE_A_ID" -- --effort high
herdr agent start verify-b --kind opencode --pane "$PANE_B_ID" -- -m opencode/mimo-v2.5-free --agent plan

herdr agent prompt verify-a "$(cat /tmp/verify_prompt.md)" --wait --timeout 600000
herdr agent prompt verify-b "$(cat /tmp/verify_prompt.md)" --wait --timeout 600000
```

Track each verification agent's `tab_id` alongside its `pane_id`/name — Phase
9 needs the `tab_id` to close it out.

Wait for both to complete (poll `herdr agent get <name>` for `agent_status:
idle` if `--wait` times out on a long-running one — this is normal for a
substantial diagnosis, not a failure). Collect their reports (`herdr agent
read <name> --source recent-unwrapped --lines 3000 --format text`; if the
pane's own scrollback is truncated, ask the agent to write its findings to a
file under `docs/` and read that instead).

Note the tab/pane/agent identifiers Herdr assigns to each of these two
verification agents (`herdr agent list` / `herdr tab list`) — you'll need the
`tab_id`s in Phase 9 to close these agents out once the card is pushed.

**If an agent is out of credits / billing-failed** (pane output or
`herdr agent read` shows a billing/quota error, or the call errors/times out
with no real response): **do not auto-retry or silently fall back.** Tell the
user which agent/kind failed.

- **If the failed agent is Grok, or an OpenCode-kind agent (Big Pickle, MiMo,
  Nemotron)** — hit its usage/rate limit, or burned through its available
  tokens/quota: say so explicitly, then use `AskUserQuestion` with
  **`Google Antigravity — fallback for <failed agent> (recommended)`**
  (`agy` kind) as the first option, alongside the usual next-best
  Routing-table alternative(s) for that task type and `Skip this verification
  slot`. This is a designated, named fallback — not a generic "pick anything"
  choice — but it is still never applied silently; the user confirms it via
  the question like any other re-dispatch.
- **If the failed agent is Claude**, there is no designated single fallback
  (Claude has no free/quota-limited sibling in this table) — offer the
  next-best alternative(s) from the Routing table as before.

Re-dispatch to whichever the user picks before moving to 4.3.

#### 4.3 Reconcile findings

Merge both reports into a single deduplicated analysis:
- **Agreed concerns** (both agents flagged) → must address.
- **Divergent opinions** → evaluate and decide; note in the card.
- **Missed edge cases** → incorporate into the design.
- **Alternative approaches suggested** → evaluate; adopt if better.

**If no Herdr / HERDR_ENV=1**: Skip to Phase 5 but note in the card that
multi-agent verification was not performed (user can request it later).

### Phase 5 — Pitch the Solution to the User (Iterative Approval)

Present the **refined solution** (incorporating Phase 4 findings) to the user as
a clear, conversational pitch — not a formal doc yet. Include:

1. **What changes** (files, functions, new code).
2. **Why this approach** (trade-offs, convention alignment).
3. **Verification plan** (exact commands, expected test count).
4. **Risks / open questions** (from Phase 4 or your design).
5. **Non-goals** (explicitly out of scope).

Ask the user to react: **Approve as-is**, **Request changes**, or **Ask for an
alternative**. Iterate here — refine the design based on feedback — until the
user explicitly approves.

**Use `AskUserQuestion` with explicit options:**
- `Approve — proceed to card drafting`
- `Request changes` (user describes adjustments)
- `Show me an alternative approach`

### Phase 6 — Draft the Final Card (Using the Template)

Once the user approves the solution, write the complete card using the template
below. Fill every applicable section; omit ones that genuinely don't apply.
Keep it dense but readable. **Every file reference must have a line anchor.**

**The `## Verification Plan` section is mandatory, not optional** — never
substitute it with just "run the test suite" when the task touches anything a
generic test/lint/build command can't observe (infra state, external
resources, DB rows, migrations, third-party config). Write it so a builder
with zero conversation context — including the `worker` skill executing this
card later — can mechanically confirm each Acceptance Criterion is actually
met, not just that the build didn't break.

#### Task Card Template

```
<TITLE — one line, imperative verb first, e.g. "Add card-skin system with
per-skin assets and a Settings selector">

<CONTEXT — 1-3 sentences: why this task exists, current behavior in plain words,
and what the acceptance hinges on. If gated on another card, say so.>

<Universiteit Utrecht cards only — mandatory: **Repo:** `<absolute path, e.g.
/home/uu/tf-azure-aipf-deployment>` — this becomes the card's `repo:` UDA at
push time (Phase 8) and is how `worker` knows which checkout to apply changes
in. Omit this line entirely for Guiñotazo cards.>

## Current State (verified)

<What is true in the code TODAY, anchored to real paths/lines. Facts only:
- `file.ts:NN` — what it contains / does today.
- Currently X does Y; the mechanism is Z.
- Live test count: N/N (verified by running the confirmed verification command).
- Verification command output: <paste key lines from the live run>
Do not suggest fixes here — just pin the starting point.>

## Goals

1. <outcome 1 — what the user sees or what works after>
2. <outcome 2>
3. <...>

## Implementation Guidance

<The "how" — as exact as possible so the builder needs no context:
- file.ts:NN — change <function/element> to ...
- Reuse the existing <pattern/helper/module> at <path> rather than inventing a new one.
- Respect <existing settings/reducer/actions mechanism> at <path>.
- Keep <module> pure/untouched (additive exports only via <index>); no GameState changes.
- Call out tricky timing/order dependencies explicitly (e.g. "this runs before X; do not reorder").>

## Constraints / Non-Goals

- Do NOT touch <files/modules>.
- Out of scope for this card: <secondary idea> (separate follow-up card).
- No new dependencies unless needed; prefer <existing stack feature>.
- Engine purity: bots/UI must not mutate src/engine; additive exports only via src/engine/index.ts.

## Acceptance Criteria

- <observable, verifiable criterion 1>
- <observable, verifiable criterion 2>
- All project verification commands clean (quote the confirmed command verbatim,
  e.g. `npm run lint && npm run test && npm run build`).
- Existing tests stay <N>/<N> (live count from Phase 1).
- Keep changes uncommitted unless told otherwise.

## Verification Plan

<MANDATORY — one concrete, executable step per Acceptance Criterion above, in
the same order, so a builder (or the `worker` skill) can mechanically confirm
"done" without re-deriving what "done" means. A single generic test/lint/build
command is NOT sufficient on its own whenever the task also changes external
or stateful things (infra resources, data migrations, DB rows, config that
isn't covered by the test suite) — spell those checks out individually too.
For each criterion give:
- **Check**: the exact command/query/manual step to run.
- **Expected result**: what a pass looks like (exact value, exit code, diff
  shape, absence of X, etc.) — not just "it works".
- **Who runs it**: note explicitly if this step is unsafe/expensive/needs
  elevated access and must be handed to the user rather than run by an agent.
  Name *which* skill defines the handoff protocol, not just that one exists —
  a fresh builder with zero context can't chase an unnamed reference (e.g.
  "see the `worker` skill's Terraform Plan/Apply Protocol for
  `terraform plan/apply` against live state", not just "per protocol"), or
  anything needing IAM/RBAC the builder may not hold.

Example (infra task):
1. Criterion "state blob renamed without data loss" → Check:
   `az storage blob show --account-name X --container-name Y --name <old>`
   and `...--name <new>` → Expected: both exist, same `contentMd5`. Runs: agent
   (read-only, safe).
2. Criterion "terraform reads the new key cleanly" → Check: `terraform init
   -backend-config=... -reconfigure && terraform plan` → Expected: plan shows
   zero unexpected resource changes. Runs: **user** — see the `worker` skill's
   "Terraform Plan/Apply Protocol" section; never run plan/apply directly
   against Universiteit Utrecht state, hand the exact command to the user
   instead.
>

<optional>
## Parent / Depends On

- Parent card: <uuid or title if subtask>
- Gated on: <card title / uuid> landing first.
</optional>

## Verification Notes (for the builder)

- Multi-agent verification: <Performed / Not performed — HERDR_ENV=1 missing>
- Task type classification: <e.g. "multi-file implementation">
- Agents used (routed): <e.g. Big Pickle (primary) + Nemotron 3 Super (cross-check) / N/A>
- Key findings incorporated: <summary of Phase 4 reconciliation>
- Verification command to run: <exact command>
```

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
   Agent/Model Routing table using the Phase 3 task-type classification (e.g.
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
spun up — Phase 1's repo-scan helpers as well as Phase 4's verification pair —
they've served their purpose and shouldn't linger:

```bash
herdr agent list         # confirm every tab/pane/agent from Phase 1 and Phase 4 (by id/name)
herdr tab close <tab_id>   # once per agent's tab spun up in Phase 1 or Phase 4
```

Do this for **every** agent started in Phase 1 or Phase 4 (skip whichever
phase's agents don't apply — e.g. Phase 4 was skipped because `HERDR_ENV=1`
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
- **Verification pedigree** (multi-agent findings, or note that it was skipped)

And the user **always** signs off at two gates: solution pitch (Phase 5) and
final card (Phase 7).

---

## Usage as a command

When invoked as `/kanban_task`, execute the full workflow above. The skill is
interactive — it will ask questions at Phase 2 (essentials, then the 2-4 round
co-design loop), Phase 5, and Phase 7 via `AskUserQuestion` or open
conversation. Do not skip any phase.

---

## Notes for the Agent Running This Skill

- **Never guess file paths** — read them first (use `read`/`grep`/`glob`).
- **Never assume tooling** — detect and confirm with the user (Phase 1).
- **Every agent this skill starts — Phase 1's repo-scan helpers included, not
  just Phase 4's verification pair — is picked from the Agent/Model Routing
  table and started via `herdr agent start`.** The generic `Agent` tool is
  never the right call anywhere in this workflow.
- **Never lock in an idea before challenging it** — Phase 2.2's co-design loop
  (~2-4 rounds) happens before Phase 4's paid Herdr agents ever spin up; don't
  skip straight from "essentials gathered" to a silent internal design.
- **Never push without approval** — two explicit gates (Phase 5, Phase 7),
  on top of Phase 2.2's earlier idea-level sign-off.
- **Verification agents never write files** — Phase 4 is diagnosis-only:
  OpenCode-kind verifiers always launch with `--agent plan` regardless of
  what the Routing table's row says (that persona is for the eventual
  builder, not the verifier), and the 4.1 prompt always includes the
  "DO NOT MODIFY ANY FILES" line for every agent kind.
- **Herdr verification is mandatory when available** — if `HERDR_ENV=1` is set
  but Herdr fails, surface the error and ask the user whether to proceed
  without it or retry.
- **Keep changes uncommitted** — the card's acceptance criteria includes this.
- **Anchor everything** — every claim about current code must have a `file:line`
  reference from your Phase 1 scan.
- **Close Herdr agents after the push** — Phase 4 spins up Herdr verification
  agents; once Phase 8 confirms the card is pushed, close those agents/panes
  (Phase 9) so they don't linger unused.
- **Every agent — Phase 1 helpers and Phase 4 verifiers alike — gets its own
  brand-new tab, never a pane split off the orchestrator, never reused** — no
  exceptions; this is also why every one of them gets closed **in full**
  (`herdr tab close`) in Phase 9 (nothing there was ever the user's own pane
  or tab).