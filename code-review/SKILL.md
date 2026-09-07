---
name: code-review
description: Comprehensive code review — scans for dead code, vulnerabilities (HIGH/MEDIUM), and best practices. Returns itemized findings with evidence. Use when you want to audit code quality and optionally create kanban tasks from findings.
---

# Code Review — Cleanup + Vulnerabilities + Best Practices → Kanban Tasks

A comprehensive code review that:
1. **Scans for dead/unused files & folders** (cleanup candidates)
2. **Finds vulnerabilities** (HIGH/MEDIUM — tfsec, npm audit, language-specific)
3. **Checks best practices** (language/framework conventions, patterns)
4. **Returns an itemized findings list** with severity, evidence, and fix estimates
5. **Lets you select which to fix** → spawns `/kanban-task` for each selected item

**Cardinal rule: report first, act only on your selection. Every finding has evidence.**

---

## Prerequisites

- Repo root is the working directory.
- `scripts/backlog` available (for kanban-task integration).
- Kanban board running (`/setup_kanban_board` if needed).
- `HERDR_ENV=1` optional but recommended for kanban-task's multi-agent verification.

---

## Workflow

### Phase 0 — Establish Baseline

```bash
git status                    # uncommitted changes = don't touch those files
git rev-parse --is-inside-work-tree
```

Detect stack/tooling (signals → verification command):

| Signal | Stack | Verification Command |
|--------|-------|---------------------|
| `package.json` | Node/TS | `npm run lint && npm run test && npm run build` |
| `go.mod` | Go | `go vet ./... && go test ./...` |
| `Cargo.toml` | Rust | `cargo check && cargo test` |
| `pyproject.toml` / `requirements.txt` | Python | `ruff check . && mypy . && pytest` |
| `*.tf` + `versions.tf` | Terraform | `terraform fmt -check && terraform validate && tfsec . --minimum-severity HIGH` |

**Run verification command once** to capture live baseline (test count, lint status).

Read `AGENTS.md` for repo-specific constraints (engine purity, etc.).

---

### Phase 1 — Cleanup Scan (Dead Files/Folders)

**Evidence-based only — no vibes.** For each candidate, record *why* it looks unused.

#### 1.1 Orphaned Source Files
- Find all exported modules/components/functions.
- `grep -r` for each export name across **entire repo** (including tests, configs, scripts, barrel files).
- Zero inbound references = candidate.
- Tools: `knip` (TS/JS), `vulture` (Python), `deadcode` (Go), `ucm` (Unison) if available.

#### 1.2 Dead Code Inside Live Files
- Exported functions/types with zero imports.
- Commented-out blocks, `if (false)` branches, feature-flag dead code.
- Language tools: `ts-prune`, `knip`, `vulture`, `golangci-lint` (unused).

#### 1.3 Stale Artifacts
- `*.bak`, `*.orig`, `*.old`, `*_backup*`, `tmp_*`, `scratch_*`, editor swaps.
- Merge conflict leftovers (`<<<<<<<` markers).
- Obvious draft pairs: `foo.tsx` + `foo.old.tsx` where only one is imported.

#### 1.4 Superseded Docs/Plans
- Design docs explicitly replaced by newer ones.
- Plan docs for shipped features (flag as "archive candidate," not auto-delete).
- **Always ask before touching docs** — they often have historical value.

#### 1.5 Unused Dependencies
- `depcheck` / `knip` (Node), `go mod tidy` (Go), `pip-autoremove` / `pipdeptree` (Python), `cargo machete` (Rust).
- Verify transitive/peer deps not falsely flagged.

#### 1.6 Committed Build Artifacts
- `dist/`, `build/`, `.next/`, `node_modules/`, `.DS_Store`, `*.log` committed.
- Fix = add to `.gitignore` + `git rm --cached`.

**Output per candidate**: `{ path, category, evidence, confidence: certain|probable|worth-asking, estimated_effort: S|M|L }`

---

### Phase 2 — Vulnerability Scan (HIGH/MEDIUM)

Run scanners appropriate to the stack. **Filter to HIGH + MEDIUM only** (LOW = noise).

| Stack | Tool | Command |
|-------|------|---------|
| Terraform | tfsec | `tfsec . --minimum-severity MEDIUM` |
| Node/JS/TS | npm audit | `npm audit --audit-level=moderate` |
| Node/JS/TS | Snyk (if configured) | `snyk test --severity-threshold=medium` |
| Python | pip-audit / bandit | `pip-audit -r requirements.txt`, `bandit -r . -ll` |
| Go | govulncheck | `govulncheck ./...` |
| Rust | cargo audit | `cargo audit` |
| Docker | Trivy | `trivy fs . --severity HIGH,MEDIUM` |
| General | GitLeaks (secrets) | `gitleaks detect --source .` |

**Cross-check against git diff** (if reviewing a PR/branch):
- Findings on **unchanged code** = pre-existing → label `pre-existing`.
- Findings on **changed code** = introduced or exposed → label `new-or-exposed`.

**Output per finding**: `{ file:line, tool, rule_id, severity: HIGH|MEDIUM, description, fix_hint, status: new-or-exposed|pre-existing }`

---

### Phase 3 — Best Practices Review

Review changed files (or whole repo if full review) for:

- **Language/framework idioms** — non-idiomatic patterns, missed stdlib features.
- **Consistency** — matches existing patterns in this repo (naming, structure, error handling).
- **Architecture** — proper layering, no circular deps, engine purity (guiñote: `src/engine` additive exports only).
- **Testing** — new code paths covered, existing tests still valid, no test-only code in prod.
- **Security hygiene** — no secrets, safe deserialization, input validation at boundaries.
- **Maintainability** — unnecessary complexity, premature abstraction, DRY violations.
- **Terraform specifics** — module structure, variable validation, sensitive values, provider pinning, blast radius.

**Output per suggestion**: `{ file:line, category, description, recommended_change, impact: high|medium|low, effort: S|M|L }`

---

### Phase 4 — Consolidate & Present Findings

Merge all findings into a **single ranked list** (most severe/actionable first):

```
┌─ FINDINGS (select which to fix) ────────────────────────────────────────┐
│ # │ Type                  │ Severity │ File:Line        │ Summary                              │ Effort │
├───┼───────────────────────┼──────────┼──────────────────┼──────────────────────────────────────┼────────┤
│ 1 │ Vuln (tfsec)          │ HIGH     │ main.tf:42       │ S3 bucket public read                │ S      │
│ 2 │ Cleanup (orphan)      │ —        │ src/old/Widget.tsx│ Zero imports, 345 LOC                │ M      │
│ 3 │ Vuln (npm audit)      │ MEDIUM   │ package.json     │ lodash prototype pollution (CVE-...) │ S      │
│ 4 │ Best Practice         │ medium   │ src/api/client.ts│ Inconsistent error handling pattern  │ S      │
│ 5 │ Cleanup (artifact)    │ —        │ dist/            │ Committed build output (gitignore)     │ S      │
│ 6 │ Best Practice         │ low      │ src/utils.ts     │ Use Array.flatMap instead of reduce  │ S      │
└───────────────────────────────────────────────────────────────────────────┘
```

**Present via `AskUserQuestion` with multi-select:**

Options = each finding row (label = `#N Type: Summary`, description = full detail + evidence).
User selects any subset (or "None — just report").

---

### Phase 5 — Create Kanban Tasks for Selected Findings

**For each selected finding**, invoke the `/kanban-task` skill (or its programmatic equivalent) with a pre-filled context:

#### Mapping: Finding → Kanban Task Input

| Finding Type | Kanban Task Title | Context Injected |
|--------------|-------------------|------------------|
| Vuln (tfsec) | `Fix tfsec HIGH: <rule> in <file>` | File:line, rule, fix_hint, verification: `tfsec` |
| Vuln (npm audit) | `Fix npm audit MEDIUM: <pkg> <CVE>` | Package, CVE, upgrade version, verification: `npm audit` |
| Cleanup (orphan) | `Remove dead code: <file>` | File, evidence (grep results), verification: full test suite |
| Cleanup (artifact) | `Remove committed build artifacts + gitignore` | Paths, `.gitignore` fix, verification: `git status` clean |
| Best Practice | `Refactor: <description> in <file>` | File:line, current vs recommended pattern, verification: lint+test |

#### Per-Task Kanban Flow

1. **Draft card** using `kanban-task` template (Phase 1-3 of that skill) — pre-filled with finding details.
2. **Present card for approval** (kanban-task Phase 5-7) — you approve each.
3. **On approval** → pushed to board with your chosen agent + priority.

**You control per-task**: agent (`claude`/`big-pickle`/`mimo`/unassigned), priority (H/M/L).

**Batch option**: If you select multiple findings of same type (e.g., 3 npm audit fixes), confirm whether to batch into one card or separate cards.

---

### Phase 6 — Summary Report

After all selected tasks are pushed:

```
✅ Code Review Complete — <N> findings scanned, <M> selected, <K> cards pushed

Pushed to Kanban:
- #123 [HIGH] Fix tfsec HIGH: S3 bucket public read (agent: claude)
- #124 [MEDIUM] Fix npm audit MEDIUM: lodash CVE-2024-XXXX (agent: big-pickle)
- #125 [LOW] Remove dead code: src/old/Widget.tsx (agent: unassigned)

Board: http://127.0.0.1:8787/
```

Unselected findings remain in the report for future reference.

---

## Usage as a Command

When invoked as `/code-review`:

1. **Scope**: Ask if reviewing current diff, a branch, a PR, or whole repo (default: whole repo).
2. **Run Phases 0-4** automatically.
3. **Phase 4**: Present interactive multi-select findings table.
4. **Phase 5**: For each selection, run `/kanban-task` sub-flow (you approve each card).
5. **Phase 6**: Final summary.

**Options** (pass as args or asked interactively):
- `--scope <diff|branch|pr|all>` — default `all`
- `--severity <high|medium|low>` — minimum severity to include (default `medium` for vulns, all for cleanup/bp)
- `--auto-batch` — batch same-type findings into single cards (ask per batch)
- `--no-kanban` — just report, don't create tasks

---

## Notes for the Agent Running This Skill

- **Never guess** — run tools, grep, read files for evidence.
- **Never delete** — cleanup findings are *candidates*; kanban-task will handle removal with verification.
- **Anchor everything** — every finding must have `file:line` and concrete evidence.
- **Pre-existing vs new** — always label vuln findings relative to the diff scope.
- **Confidence matters** — low-confidence cleanup items flagged as `worth-asking`, not bundled.
- **Engine purity** — for guiñote, any finding touching `src/engine` must note: "additive exports only via `src/engine/index.ts`; no GameState mutations."
- **Verification command** — detect once (Phase 0), use for all kanban-task acceptance criteria.
- **HERDR_ENV=1** — if set, kanban-task will run multi-agent verification; if not, note in card.