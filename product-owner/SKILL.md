---
name: product-owner
description: Kanban board maintenance with three modes: pitch (propose ideas), groom (board health), author (write cards). Use for strategic backlog management and creating executable task cards.
---

# Product Owner — Board Maintenance + Idea Pitching + Card Authoring

A strategic product ownership skill that **maintains the Kanban board health**,
**pitches prioritized ideas** based on repo reality, and **authors executable
cards** (the original card-writing workflow). Three modes in one:

| Mode | Command | Purpose |
|------|---------|---------|
| **Pitch** | `/product-owner pitch` | Scan repo + board → propose 3-5 high-impact ideas with rationale |
| **Groom** | `/product-owner groom` | Board health: stale cards, duplicates, priority drift, dependency ordering |
| **Author** | `/product-owner author` | Original workflow: raw idea → detailed card → approval → push |

**Cardinal rule: every action touches the board only with your explicit approval.**

**Universiteit Utrecht cards require a `repo:` field.** That board fans out
across every `tf-*` repo in the home directory (see this repo's CLAUDE.md), so
a card with no repo recorded is unusable by `worker` later — it has no way to
know which checkout to apply changes in. Whenever `BACKLOG_PROJECT` resolves
to Universiteit Utrecht (check the exported env var, or ask if unset and the
current repo looks like a `tf-*` Terraform/Azure repo), Author mode must
resolve the absolute repo path and pass it through as `--repo` at push time
(step 9). This does not apply to Guiñotazo or other single-repo boards.

---

## Prerequisites

- `scripts/backlog` in repo root.
- Kanban board at `http://127.0.0.1:8787/` (run `/setup_kanban_board` if needed).
- `AGENTS.md` read for constraints (engine purity, verification commands).
- Herdr CLI + `HERDR_ENV=1` — optional, enables Author mode's multi-agent
  verification step (Grok + Claude). Author mode works without it; that step
  is just skipped and noted in the card.

---

## Shared Context (Loaded in Every Mode)

Before any mode runs, silently gather:

```bash
# Board state
bash scripts/backlog board
bash scripts/backlog next

# Repo health
git status
git log --oneline -20
npm run lint && npm run test && npm run build   # or detected verification cmd

# Repo structure (key dirs, engine purity rules, test patterns)
# AGENTS.md constraints
```

**This context informs every pitch/groom/author decision.**

---

## Mode 1: Pitch — `/product-owner pitch`

**Goal:** Propose 3-5 concrete, high-impact ideas you could author as cards.

### Pitch Generation Process

1. **Analyze signals** from shared context:
   - **Tech debt**: failing tests, lint warnings, outdated deps, `TODO`/`FIXME` density.
   - **Feature gaps**: missing tests for core engine, settings not persisted, no CI.
   - **UX polish**: missing animations, keyboard shortcuts, mobile layout, sound.
   - **Architecture**: engine purity violations, circular deps, barrel file bloat.
   - **Board state**: cards stuck in `active` > 3 days, no `review` movement, priority inversions.

2. **Generate candidate ideas** (internal, 8-12 raw):
   - Mix of: features, fixes, refactors, tech debt, docs, infra.
   - Each tagged: `impact: high|med|low`, `effort: S|M|L`, `type: feature|fix|refactor|debt|docs|infra`.

3. **Select & refine top 3-5** using **RICE-ish scoring**:
   - Reach (how many users/flows affected)
   - Impact (magnitude of improvement)
   - Confidence (how sure based on code evidence)
   - Effort (S/M/L from code scan)

4. **Present as pitched ideas** (not cards yet):

```
┌─ PRODUCT PITCHES (pick which to author) ─────────────────────────────────┐
│ # │ Title                                      │ Type     │ Impact │ Effort │
├───┼────────────────────────────────────────────┼──────────┼────────┼────────┤
│ 1 │ Add card-skin system with Settings picker  │ feature  │ High   │ M      │
│ 2 │ Fix engine purity: bot imports GameState   │ refactor │ High   │ S      │
│ 3 │ Persist settings to localStorage           │ feature  │ Med    │ S      │
│ 4 │ Add deal animation + sound                 │ polish   │ Med    │ S      │
│ 5 │ CI pipeline: lint+test on PR               │ infra    │ High   │ S      │
└────────────────────────────────────────────────────────────────────────────┘
```

**Each pitch includes:**
- One-liner rationale (anchored to code: "engine/pure.ts:42 imports GameState")
- Suggested acceptance criteria (2-3 bullets)
- Suggested agent + priority
- Dependencies/blockers (other cards, external)

### Your Action

**Ask via `AskUserQuestion` (multi-select):**
- Select any subset to **author as cards** (launches Mode 3 per selection).
- "None — just report" to keep ideas for later.
- "Generate more" to re-run with different filters (e.g., "only tech debt").

---

## Mode 2: Groom — `/product-owner groom`

**Goal:** Board hygiene — surface problems, propose fixes, execute on approval.

### Grooming Checks (Automated)

| Check | Detection | Action Proposed |
|-------|-----------|-----------------|
| **Stale active** | `active` > 72h no movement | `backlog back` (return to todo) or reassign |
| **Stale review** | `review` > 72h no human verdict | Ping human, or `backlog back` if abandoned |
| **Priority drift** | `High` cards below `Medium` in todo | Re-prioritize (ask) |
| **Duplicate/overlap** | Similar titles/descriptions | Merge or close duplicate (ask) |
| **Orphaned deps** | Card A "gated on" B but B not on board | Flag missing dependency |
| **No agent assigned** | `todo` cards with empty `agent:` | Suggest agent based on type |
| **Missing repo (Universiteit Utrecht only)** | Card on that board with empty `repo:` | Ask user for the absolute repo path, then `task <uuid> modify repo:<path>` (or `backlog repo <uuid> <path>`) |
| **Vague acceptance** | Cards without verifiable criteria | Rewrite criteria (launch Mode 3) |
| **Done not archived** | `done` cards > 7 days | `backlog archive` (ask) |

### Output

```
┌─ BOARD GROOMING REPORT ──────────────────────────────────────────────────┐
│ Issue                    │ Cards Affected        │ Proposed Action          │
├──────────────────────────┼───────────────────────┼──────────────────────────┤
│ Stale active (5d)        │ #42, #55              │ backlog back → todo      │
│ Priority inversion       │ #61 (High) below #59  │ Reorder: #61 first       │
│ No agent assigned        │ #33, #47, #58         │ Suggest: big-pickle      │
│ Vague acceptance         │ #29 "improve UI"      │ Rewrite via author mode  │
│ Done not archived (12d)  │ #12, #18, #24         │ backlog archive          │
└──────────────────────────┴───────────────────────┴──────────────────────────┘
```

**Ask via `AskUserQuestion` (multi-select):**
- Select fixes to apply (each runs the `backlog` command).
- "Show details" for any row.
- "Skip all" to just report.

---

## Mode 3: Author — `/product-owner author`

**Goal:** Original card-writing workflow (draft → approve → push).

### Workflow (Condensed from Original Skill)

1. **Clarify ask** — user provides raw idea/ticket/notes. **If the target
   board is Universiteit Utrecht, also pin down the exact absolute repo path**
   this card applies to (default: current repo root) — don't leave it vague,
   it's a hard requirement for this board, not an optional detail.
2. **Scan repo** — anchor to real files/lines (grep/read).
3. **Confirm tooling** — detect stack, run verification, confirm with user.
4. **Draft solution** — internal design (files touched, why, trade-offs, non-goals).
5. **Draft card** — use template below (every claim has `file:line`).
6. **Approval gate** — present full card, `AskUserQuestion`: `Approve` / `Request changes`.
7. **Agent + Priority** — ask (recent agents from board + `Unassigned`; H/M/L).
8. **Push** — `bash scripts/backlog add "..." --priority X --agent Y [--repo
   <absolute path>]`. `--repo` is **required** when `BACKLOG_PROJECT` is
   Universiteit Utrecht (the wrapper refuses the push without it on that
   board); omit it for Guiñotazo/other single-repo boards.
9. **Confirm** — report card ID, state, board URL.

**Author mode is the fast lane — no automated multi-agent verification runs
here.** If a draft needs independent multi-agent scrutiny before it's
written up (parallel solution proposals, synthesis, a pitch showing what was
explored and why), that's what `kanban-task` is for; running the same kind
of verification twice across two skills was pure overhead. The human
approval gate (step 6) is still mandatory either way.

### Multi-Agent Verification (Herdr) — Verifier A routed dynamically, Verifier B fixed

Verifier B is always the Claude orchestrator (below). Verifier A is **not**
fixed to one agent — pick it per run from
[`../shared/agent-routing.md`](../shared/agent-routing.md)'s routing table
(same file `kanban-task` and `worker` use), based on what the drafted
solution actually is:

1. **Classify** the drafted solution by primary task type (the shared
   table's categories), the same way `kanban-task` Phase 3 does.
2. **Pick Verifier A** from the matching row. Default to the Free/Low-cost
   pick that fits — don't reach for Grok just because it's listed; use it
   only when the row genuinely says diverse-cross-check or the draft is
   itself high-risk.
3. **Effort selection** (Grok/Antigravity only): follow the shared file's
   Effort Selection section — present the recommendation first, labeled
   `<level> — recommended (<one-line reason>)`, with 1-2 adjacent levels as
   alternatives. Recommend `low` for a simple, well-scoped pitch/fix,
   `medium` for routine card verification, `high` only when the draft is
   itself unusually high-risk. **Whichever agent Verifier A lands on,
   explicitly tell the user which one (and briefly why) before starting
   it** — don't fold it in silently.
4. **Billing-failure fallback**: follow the shared file's "Billing-Failure /
   Rate-Limit Fallback" section, substituting "verification slot" for
   "proposal slot." (Verifier B is always Claude, a fixed role, not routed
   from the shared table.)

- **Verifier B — Claude, the orchestration agent, always.** This one is allowed to
  spin up its own helper agents if the verification prompt is broad enough to
  need them (e.g. splitting "check the engine side" from "check the UI side").
  **When it does, its helpers must each get their own new Herdr pane**
  (`herdr pane split --current --direction right --cwd "$PWD" --no-focus` then
  `herdr agent start <helper-name> --kind ... --pane <id> -- ...`) — never
  spawned invisibly inline. Say this explicitly in the prompt you send to
  Verifier B, e.g.:

  > "If you need help from other agents to verify this, spin each one up in
  > its own new Herdr pane (`herdr pane split` + `herdr agent start`) rather
  > than handling it all yourself inline. Collect all of their findings
  > yourself and give me back one consolidated report — I only need your
  > final summary, not a transcript of each helper."

  This keeps every pane in the workspace individually inspectable (good for
  the user following along live) while still handing you back exactly one
  report per verifier to reconcile — Verifier B's own report already merges
  whatever its helpers found, so you reconcile two reports total (A + B), not
  N.

Both verifiers are diagnosis-only (no fixes applied). Reconcile agreed
concerns, divergent opinions, and missed edge cases into the draft before
step 6, same as `kanban-task` Phase 4.3. Close every pane opened for this
step (Verifier B's helper panes included — `herdr agent list`/`herdr pane
list` to find them) once the card is pushed.

### Task Card Template

Use the template at
[`../shared/card-template.md`](../shared/card-template.md) — same one
`kanban-task` uses. Fill every applicable section, omit ones that don't
apply, every file reference needs a line anchor. Its `## Verification Plan`
section is **mandatory**, and its `## Rollout & Blast Radius` section is
mandatory whenever this card touches a template/shared-infra repo consumed
by more than one other repo — see that file for exactly what each requires.
Its `<Universiteit Utrecht cards only>` repo line and "push time" reference
map to this mode's step 9 below.

---

## Usage as a Command

```
/product-owner pitch      # Propose ideas → select → author cards
/product-owner groom      # Board health report → select fixes → apply
/product-owner author     # Raw idea → detailed card → approval → push
/product-owner            # Default: show board summary + ask which mode
```

**Default (no subcommand):**
```
bash scripts/backlog board
# Shows: todo (N), active (N), review (N), done (N)
# Then asks: "Pitch ideas? Groom board? Author a card? View board?"
```

---

## Notes for the Agent Running This Skill

- **Never push without approval** — every board mutation (add, back, review, done, archive, assign) requires your `AskUserQuestion` confirmation.
- **Anchor everything** — pitches/grooming findings cite `file:line` or card IDs.
- **Read AGENTS.md** — bake engine purity, verification commands into every card.
- **Use live data** — run verification command at author time for real test counts.
- **Board is source of truth** — `bash scripts/backlog board` before any pitch/groom.
- **Agent assignment** — look at recent cards for actual agent names in use (`claude`, `big-pickle`, `mimo`).
- **HERDR_ENV=1** — if set, Author mode runs its own verification pair before
  drafting the card — see Mode 3: Verifier A routed per task type (table in
  Mode 3, effort confirmed only when it lands on Grok), Verifier B always the
  Claude orchestrator. If a verifier spins up helpers, those must land in new
  Herdr panes, never invisible inline subagents, so the workspace stays
  inspectable; the verifier still owes you one consolidated report, not a
  transcript per helper. Close every pane opened for this step once pushed.