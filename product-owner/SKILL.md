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
- Kanban board at `http://127.0.0.1:8787/` (run `/setup-kanban-board` if needed).
- `AGENTS.md` read for constraints (engine purity, verification commands).
- Herdr CLI + `HERDR_ENV=1` — **not needed by this skill.** All three modes
  run entirely in this session; no Herdr agents are dispatched. If a draft
  warrants independent multi-agent scrutiny before it's written up, use
  `kanban-task` instead of Author mode.

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
- **Agent assignment** — the `agent:` field is an enum; only these values are
  legal, and `backlog add --agent <x>` fails outright on anything else:
  `claude`, `big-pickle`, `mimo`, `grok`, `antigravity`, `human`,
  `orchestrator`, `codex`. Use the exact token (lowercase, hyphenated) — not
  a display label like "Big Pickle". See `../shared/agent-routing.md`.
- **No Herdr agents here** — all three modes run in this session. Independent
  multi-agent scrutiny of a draft is `kanban-task`'s job, not Author mode's.