# Shared: Task Card Template

Canonical Taskwarrior card template. Referenced by `kanban-task` (Phase 6)
and `product-owner` (Author mode, step 6) — **edit here once, not in two
places**, so a template improvement (or a bug found in it) doesn't have to
be re-applied by hand to a second copy that's drifted out of sync.

Fill every applicable section; omit ones that genuinely don't apply. Keep it
dense but readable. **Every file reference must have a line anchor.**

**The `## Verification Plan` section is mandatory, not optional** — never
substitute it with just "run the test suite" when the task touches anything
a generic test/lint/build command can't observe (infra state, external
resources, DB rows, migrations, third-party config). Write it so a builder
with zero conversation context can mechanically confirm each Acceptance
Criterion is actually met, not just that the build didn't break.

**The `## Rollout & Blast Radius` section is mandatory whenever the card
touches a template/shared-infra repo consumed by more than one other
repo** — omit it entirely for a self-contained single-repo change.

## Template

```
<TITLE — one line, imperative verb first, e.g. "Add card-skin system with
per-skin assets and a Settings selector">

<CONTEXT — 1-3 sentences: why this task exists, current behavior in plain words,
and what the acceptance hinges on. If gated on another card, say so.>

<Universiteit Utrecht cards only — mandatory: **Repo:** `<absolute path, e.g.
/home/uu/tf-azure-aipf-deployment>` — this becomes the card's `repo:` UDA at
push time and is how `worker` knows which checkout to apply changes in. Omit
this line entirely for Guiñotazo cards.>

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

<template/shared-infra repos with more than one consumer only, mandatory —
omit entirely for a self-contained single-repo fix:
## Rollout & Blast Radius

- **Affected**: <exact list of consuming repos/environments>
- **Rollout**: <version tag / opt-in flag / staged rollout mechanism — not "just merge it">
- **Rollback**: <what undoing this looks like if a consumer breaks>
>

## Acceptance Criteria

- <observable, verifiable criterion 1>
- <observable, verifiable criterion 2>
- All project verification commands clean (quote the confirmed command verbatim,
  e.g. `npm run lint && npm run test && npm run build`).
- Existing tests stay <N>/<N> (live count from discovery).
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

- Solution proposals: <Performed / Not performed — HERDR_ENV=1 missing>
- Task type classification: <e.g. "multi-file implementation">
- Agents used (routed): <e.g. Big Pickle + Grok / N/A>
- What was explored and why this won: <summary of the pitch's "What I Explored" /
  "Why It Wins", i.e. the reconciliation across proposals>
- Verification command to run: <exact command>
```
