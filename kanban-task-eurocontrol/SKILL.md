---
name: kanban-task-eurocontrol
description: Eurocontrol-environment variant of kanban-task — Claude is the only agent kind installed there (no OpenCode/Big Pickle/MiMo/Nemotron, no Grok, no Google Antigravity), and it's a single hardcoded board with no repo-name routing. Deep discovery → co-design loop with the user → Claude-only verification → pitch → approval → push. Use this instead of the regular kanban-task skill whenever the card belongs on the Eurocontrol board.
---

# Kanban Task — Eurocontrol Environment (Claude-Only)

This is the Eurocontrol-specific variant of the regular `kanban-task` skill.
**Only `claude` is installed as an agent kind in this environment** — no
OpenCode (Big Pickle / MiMo V2.5 / Nemotron), no Grok, no Google Antigravity.
Every agent dispatch in this skill uses `--kind claude`, full stop. Use the
regular `kanban-task` skill for Guiñotazo / Universiteit Utrecht work; use
this one only when the card belongs on the Eurocontrol board.

**The cardinal rule: clarify → co-design with the user → draft → verify
(Claude-only) → pitch → approve → push.** Never run `scripts/backlog add`
until the user has explicitly approved the final card.

**Hard requirement — verification agents never share the orchestrator's pane
or tab.** A verification agent must never run as a `herdr pane split` off the
orchestrator's own tab — it gets cramped/unreadable as agents pile in. Every
verification agent gets its own brand-new tab (`herdr tab create`), never a
split pane. Once its work is done, close its **entire tab**
(`herdr tab close <tab_id>`), not just its pane. See Phase 4.2 and Phase 9.

---

## Before You Begin — Context Hygiene

This skill is long and multi-phase, and Phase 4 spins up two Herdr agents —
it accumulates a lot of tokens in the main conversation. **The first thing
you do when this skill is invoked is remind the user to run `/clear` first**,
unless the conversation is already fresh. Say something like: "This is a
long-running skill — worth running `/clear` first to keep the session light.
Ready when you are." Then wait for them to clear and re-invoke, or to
explicitly say to proceed without clearing.

---

## Prerequisites

- Herdr CLI installed and `HERDR_ENV=1` in the environment. Herdr itself is
  present — it's the *other agent kinds* (OpenCode, Grok, Google Antigravity)
  that are not installed in this environment, not Herdr.
- `~/.claude/skills/scripts/backlog` wrapper, with `BACKLOG_PROJECT="Eurocontrol"`
  exported before every call.
- Taskwarrior + taskwarrior-kanban backend running (`~/.task` store).

---

## Board (One Board, No Routing)

This skill always targets a single board:

```bash
export BACKLOG_PROJECT="Eurocontrol"
```

There is no board-routing question to ask and no repo-name matching — unlike
the regular `kanban-task` skill (which juggles Guiñotazo vs. Universiteit
Utrecht), every card this skill drafts goes on the Eurocontrol board. There is
also **no `repo:` field requirement** — this environment has no multi-repo
fan-out; operate in the current working directory.

---

### Phase 0 — Ensure Kanban Board is Up

Before anything else, verify the board is reachable at `http://127.0.0.1:8787/`.
If not, run the `setup-kanban-board` skill (or equivalent steps) to start it.

### Phase 1 — Read & Understand the Repo (Silent Deep Scan)

**Do this before asking any clarifying questions.** Build a real picture:

- **Structure**: key directories/modules, how the repo is organized.
- **Stack & conventions**: languages, tools, existing patterns for the kind of
  change likely being asked for.
- **Constraints file**: read `AGENTS.md`/`CLAUDE.md` if present for any
  project-specific rules (purity boundaries, forbidden files, etc.).
- **Verification commands**: detect the project's tooling signals
  (`package.json`, `Makefile`, `requirements.txt`, etc.) and identify the
  exact verification command. **Run it once now to capture live numbers**
  (test count, lint status) — never trust docs or memory.
- **Recent direction**: `git log --oneline -20` for active work trajectory.
- **Existing backlog cards**: `BACKLOG_PROJECT=Eurocontrol bash
  ~/.claude/skills/scripts/backlog next` and `... board` to see current tasks,
  priorities, and avoid duplication.

**Output**: Keep this internal. Use it to anchor every claim in the card to
real file:line references.

### Phase 2 — Clarify the Ask, Then Co-Design the Idea With the User (2-4 Rounds)

This phase has two parts: first pull out the missing facts, then — before any
Herdr agent spins up — spend a few rounds actually thinking the idea through
*with* the user, not just at them.

#### 2.1 Clarify the essentials

Ask the user for the task. Accept free text, a pasted ticket, or rough notes.
Pull out missing essentials:

- What is the **outcome** (what should work/change when done)?
- Anything **out of scope** that should NOT be touched?
- Fresh feature, fix, refactor, or subtask of existing card?
- Any hard constraints (deadlines, blocking other work)?

#### 2.2 Co-design loop — challenge the idea before committing to it (~2-4 rounds)

Once you have enough to sketch an approach, **don't jump straight to a
locked-in design.** Think out loud with the user first — this is cheap (no
Herdr agents involved yet) and is where the biggest wins usually come from,
before Phase 3-4 sink real effort into stress-testing a design that turns out
to be the wrong one.

Each round:

1. **Present a short idea sketch** (a few sentences): the approach you're
   leaning toward, anchored to real `file:line` where relevant.
2. **Name at least one real alternative or trade-off** you considered — don't
   present one path as if it were the only option.
3. **Actively ask the user to poke holes in it** — not a yes/no rubber stamp.
   Use `AskUserQuestion` or plain conversation: "Is there anything you'd do
   differently here? A simpler way? Something I'm missing?"
4. **Incorporate whatever comes back** and refine the sketch before the next
   round.

Repeat for **roughly 2-4 rounds** — fewer if the user converges quickly
("looks good, go with it" — take that at face value and stop), more only if
round 4 still surfaced a real open question. The point is genuine convergence
on a good idea, not hitting a quota.

**Only once the user has explicitly signed off on the shape of the idea** do
you move to Phase 3 and the rest of the flow. **If the idea is clear enough
after 2.1 that there's really nothing to challenge** (a small, unambiguous
fix with one obvious approach), a single round confirming "here's the
approach, sound right?" is enough — don't manufacture disagreement.

### Phase 3 — Design a Best-Practices Solution (Internal)

Using the repo understanding (Phase 1) and the clarified/co-designed ask
(Phase 2), design a concrete implementation approach:

- **What changes** at a high level (files/modules touched, new
  resources/functions, anything removed).
- **Why this approach** over obvious alternatives — call out the trade-off.
- **How it fits existing conventions** rather than inventing new ones.
- **What it deliberately does NOT handle** (explicit non-goals).
- **Risky areas** — timing/order dependencies, test implications.

**Do not present this yet.** This is your internal design that gets
stress-tested in Phase 4.

### Phase 4 — Claude-Only Verification (Herdr CLI)

**If `HERDR_ENV=1` is available**, spin up **two independent Claude agents**
in their own brand-new Herdr tabs to critique the proposed solution in
parallel. This is a **diagnosis-only** step — neither agent applies fixes.

Since only Claude is installed here, you lose the cross-model diversity the
regular `kanban-task` skill gets from pairing e.g. Claude with an OpenCode
agent — two independent Claude sessions (ideally at different effort levels)
is the best available substitute, and still catches things a single pass
misses.

#### 4.1 Prepare the verification prompt

```
REPO: <repo root path>
TASK: <user's ask, clarified and co-designed>
PROPOSED APPROACH: <your Phase 3 design, with file:line anchors>
CONSTRAINTS: <AGENTS.md/CLAUDE.md rules, verification commands>
EXISTING PATTERNS: <key patterns from Phase 1 to follow>
VERIFICATION COMMAND: <exact command to run>

DO NOT MODIFY ANY FILES. This is diagnosis only — critique the proposed
approach (soundness, edge cases, better alternatives) and report back. Do not
implement, fix, or refactor anything, even if the fix looks trivial.
```

**Always include the "DO NOT MODIFY ANY FILES" line verbatim** — Claude's
`claude` Herdr kind has no read-only persona (unlike OpenCode's `plan`), so
this prompt instruction is the only guardrail against a verifier just
implementing the change instead of critiquing it.

#### 4.2 Choose effort, then dispatch two Claude agents

Before starting either agent, propose a recommended `--effort` level
(`low`, `medium`, `high`, `xhigh`, `max`) based on the task's actual
complexity, and confirm with the user via `AskUserQuestion` (one question per
agent, batched into a single call if possible). Recommend `medium` for a
routine, well-scoped design, `high` for genuine architecture/correctness
risk, `xhigh`/`max` only for something Phase 3 flagged as unusually
high-risk. Consider giving the two verifiers **different** effort levels
(e.g. one `medium`, one `high`) rather than identical ones — that at least
varies how thoroughly each one reasons, a partial substitute for the missing
cross-model diversity.

**Never split a pane off the orchestrator's tab, and never reuse an idle
pane.** `herdr tab create` per agent, every time, no exceptions:

```bash
TAB_A_JSON=$(herdr tab create --cwd "$PWD" --no-focus)
TAB_A_ID=$(echo "$TAB_A_JSON" | jq -r '.result.tab.tab_id')
PANE_A_ID=$(echo "$TAB_A_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start verify-a --kind claude --pane "$PANE_A_ID" -- --effort <level>

TAB_B_JSON=$(herdr tab create --cwd "$PWD" --no-focus)
TAB_B_ID=$(echo "$TAB_B_JSON" | jq -r '.result.tab.tab_id')
PANE_B_ID=$(echo "$TAB_B_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start verify-b --kind claude --pane "$PANE_B_ID" -- --effort <level>

herdr agent prompt verify-a "$(cat /tmp/verify_prompt.md)" --wait --timeout 600000
herdr agent prompt verify-b "$(cat /tmp/verify_prompt.md)" --wait --timeout 600000
```

Track each verification agent's `tab_id` — Phase 9 needs it to close the tab.

Wait for both to complete (poll `herdr agent get <name>` for `agent_status:
idle` if `--wait` times out — normal for a substantial diagnosis). Collect
their reports (`herdr agent read <name> --source recent-unwrapped --lines
3000 --format text`; if scrollback is truncated, ask the agent to write its
findings to a file and read that instead).

**If an agent is out of credits / billing-failed**: stop, tell the user which
one failed. **There is no fallback agent kind in this environment** — do not
suggest Grok, Google Antigravity, or any OpenCode model, none of them are
installed here. Offer only, via `AskUserQuestion`: `Retry the same agent
later` or `Skip this verification slot`.

#### 4.3 Reconcile findings

Merge both reports into a single deduplicated analysis:
- **Agreed concerns** (both agents flagged) → must address.
- **Divergent opinions** → evaluate and decide; note in the card.
- **Missed edge cases** → incorporate into the design.
- **Alternative approaches suggested** → evaluate; adopt if better.

**If no Herdr / HERDR_ENV=1**: Skip to Phase 5 but note in the card that
multi-agent verification was not performed.

### Phase 5 — Pitch the Solution to the User (Iterative Approval)

Present the **refined solution** (incorporating Phase 4 findings) to the user
as a clear, conversational pitch:

1. **What changes** (files, functions, new code).
2. **Why this approach** (trade-offs, convention alignment).
3. **Verification plan** (exact commands, expected test count).
4. **Risks / open questions** (from Phase 4 or your design).
5. **Non-goals** (explicitly out of scope).

**Use `AskUserQuestion` with explicit options:**
- `Approve — proceed to card drafting`
- `Request changes` (user describes adjustments)
- `Show me an alternative approach`

Iterate here — refine the design based on feedback — until the user
explicitly approves.

### Phase 6 — Draft the Final Card (Using the Template)

Once the user approves the solution, write the complete card using the
template below. Fill every applicable section; omit ones that genuinely
don't apply. **Every file reference must have a line anchor.**

**The `## Verification Plan` section is mandatory, not optional** — never
substitute it with just "run the test suite" when the task touches anything a
generic test/lint/build command can't observe. Write it so a builder with
zero conversation context — including the `worker-eurocontrol` skill
executing this card later — can mechanically confirm each Acceptance
Criterion is actually met.

#### Task Card Template

```
<TITLE — one line, imperative verb first>

<CONTEXT — 1-3 sentences: why this task exists, current behavior in plain
words, and what the acceptance hinges on. If gated on another card, say so.>

## Current State (verified)

<What is true in the code TODAY, anchored to real paths/lines. Facts only:
- `file.ts:NN` — what it contains / does today.
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
- Call out tricky timing/order dependencies explicitly.>

## Constraints / Non-Goals

- Do NOT touch <files/modules>.
- Out of scope for this card: <secondary idea> (separate follow-up card).
- No new dependencies unless needed; prefer <existing stack feature>.

## Acceptance Criteria

- <observable, verifiable criterion 1>
- <observable, verifiable criterion 2>
- All project verification commands clean (quote the confirmed command verbatim).
- Existing tests stay <N>/<N> (live count from Phase 1).
- Keep changes uncommitted unless told otherwise.

## Verification Plan

<MANDATORY — one concrete, executable step per Acceptance Criterion above, in
the same order, so a builder (or the `worker-eurocontrol` skill) can
mechanically confirm "done". For each criterion give:
- **Check**: the exact command/query/manual step to run.
- **Expected result**: what a pass looks like (exact value, exit code, diff
  shape, absence of X, etc.) — not just "it works".
- **Who runs it**: note explicitly if this step is unsafe/expensive/needs
  elevated access and must be handed to the user rather than run by an agent.>

<optional>
## Parent / Depends On

- Parent card: <uuid or title if subtask>
- Gated on: <card title / uuid> landing first.
</optional>

## Verification Notes (for the builder)

- Multi-agent verification: <Performed / Not performed — HERDR_ENV=1 missing>
- Agents used: <e.g. Claude (verify-a, effort medium) + Claude (verify-b, effort high) / N/A>
- Key findings incorporated: <summary of Phase 4 reconciliation>
- Verification command to run: <exact command>
```

### Phase 7 — Final Approval Gate (Card Push)

Present the **complete drafted card** in the chat. **Do NOT push yet.**

Use `AskUserQuestion` (single call, up to 4 questions) to get:

1. **Approve card?** — Options: `Approve — push it as drafted` (recommended),
   `Request changes` (user describes adjustments; loop back to Phase 6).
2. **Which agent should build it?** — Since only Claude is installed in this
   environment, offer only `Claude — recommended` and `Unassigned — leave
   agent blank`. Do not offer Big Pickle/MiMo/Nemotron/Grok/Antigravity —
   none of them run here.
3. **What priority?** — Options: `High` (blocking/urgent), `Medium` (normal
   backlog work), `Low` (nice-to-have). Don't default silently.

**Only on "Approve" → proceed to Phase 8.**

### Phase 8 — Push to the Board

```bash
export BACKLOG_PROJECT="Eurocontrol"
bash ~/.claude/skills/scripts/backlog add "<full card text>" --priority <H|M|L> [--agent claude]
```

- `--priority`: H/M/L from user's choice.
- `--agent`: `claude` if chosen; **omit entirely** if "Unassigned" was picked.
- No `--repo` flag — this board has no repo-field requirement.

Confirm back to the user:
- New card ID, state `todo`, assigned builder (`agent:`).
- Board URL: `http://127.0.0.1:8787/`.
- Commands to pick it up: `BACKLOG_PROJECT=Eurocontrol bash
  ~/.claude/skills/scripts/backlog claim`, then `review`/`done`.

### Phase 9 — Clean Up the Herdr Verification Agents

Once the card push is confirmed, close out the Herdr agents spun up for
Phase 4 verification:

```bash
herdr agent list         # confirm the tabs/panes/agents from Phase 4
herdr tab close <tab_id>   # once per verification agent's tab
```

Do this for **every** verification agent started in Phase 4 (skip if Phase 4
was skipped because `HERDR_ENV=1` was unavailable). Confirm to the user that
the verification agents have been closed.

---

## Quality Bar — The Card Must Enable Zero-Context Execution

An agent should be able to hand this card to a fresh session and see exactly:
what to build, where, how, what to avoid, when it's finished, and how to
confirm it's finished — the per-criterion `Verification Plan` is the
mechanical check for that last part, not just "the test suite passed".

And the user **always** signs off at two gates: solution pitch (Phase 5) and
final card (Phase 7) — on top of Phase 2.2's earlier idea-level sign-off.

---

## Usage as a command

Invoke as `kanban-task-eurocontrol` whenever the card belongs on the
Eurocontrol board — never the regular `kanban-task` skill for this
environment, since that one assumes agent kinds (OpenCode/Grok/Antigravity)
that aren't installed here. Interactive at Phase 2 (essentials + co-design
loop), Phase 5, and Phase 7. Do not skip any phase.

---

## Notes for the Agent Running This Skill

- **Never guess file paths** — read them first (use `read`/`grep`/`glob`).
- **Never assume tooling** — detect and confirm with the user (Phase 1).
- **Never lock in an idea before challenging it** — Phase 2.2's co-design
  loop (~2-4 rounds) happens before Phase 4's Claude agents spin up.
- **Never push without approval** — two explicit gates (Phase 5, Phase 7).
- **Claude is the only agent kind here** — never dispatch OpenCode, Grok, or
  Google Antigravity; they are not installed in this environment. If Claude
  itself fails (credits/rate limit), there is no fallback — stop and tell
  the user.
- **Verification agents never write files** — the 4.1 prompt's "DO NOT
  MODIFY ANY FILES" line is the only guardrail (Claude's Herdr kind has no
  read-only persona to fall back on), so never omit it.
- **Keep changes uncommitted** — the card's acceptance criteria includes this.
- **Anchor everything** — every claim about current code must have a
  `file:line` reference from your Phase 1 scan.
- **Close Herdr agents after the push** — Phase 4 spins up two Claude
  verification agents; once Phase 8 confirms the card is pushed, close those
  agents/tabs (Phase 9) so they don't linger unused.
- **Every verification agent gets its own brand-new tab, never a pane split
  off the orchestrator, never reused** — no exceptions.
