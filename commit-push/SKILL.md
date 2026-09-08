---
name: commit-push
description: Commit, push, and open the Azure DevOps PR for the current working-tree changes, deriving the branch name instead of asking for it. Same gated flow as make_pr (fmt, blocking lint/tfsec/validate gate, .gitignore check, deliberate staging, mandatory secret scan, confirmed push, az repos pr create). Use when the user wants the change shipped without being asked to name the branch — e.g. after a review or an agent's fixes.
---

# Commit & Push

**This is `make_pr` with one deliberate difference: the branch name is
derived, not asked for.** Everything else — the gates, the safety rules, the
Azure DevOps PR creation — is identical, so it lives in exactly one place
rather than being maintained twice.

Follow [`../make_pr/SKILL.md`](../make_pr/SKILL.md) start to finish, with
these two deltas:

1. **Step 1 (branch name): don't ask — derive it.** Take the name from the
   nature of the diff you saw in Step 0 (kebab-case, prefixed `fix/`,
   `feat/`, or `chore/`, e.g. `fix/acr-public-network-access`). Show the
   derived name to the user as part of the Step 9 push/PR confirmation
   instead of raising a separate question up front. If the current branch is
   already a non-`main` branch that matches this change, stay on it.
   **Exception:** if the diff is ambiguous enough that a wrong guess would be
   confusing, fall back to `make_pr`'s Step 1 and just ask.
2. **Step 10.3 (PR description): reuse existing context if there is any.** If
   this change came from a kanban card (via `worker`), pull that card's
   context into the description rather than writing generic filler — read it
   with `task <UUID> export` — so the reviewer gets what the implementing
   agent had. Otherwise summarize the diff and what was verified, as
   `make_pr` says.

Everything else is unchanged and **not optional** — in particular the
blocking lint/tfsec/validate gate (Step 3), the `.gitignore` sanity check
(Step 4), name-by-name staging (Step 6), the mandatory pre-push secret scan
(Step 8), and explicit user confirmation before the push and the PR (Steps
9-10). Local reversible steps proceed without asking; remote-visible steps
never do.

## Why this is a pointer and not a copy

These two skills were maintained as near-identical 100+ line copies and
drifted: this one's Azure DevOps remote parsing never learned the
`git@ssh.dev.azure.com:v3/<org>/<project>/<repo>` SSH form that every repo in
this org actually uses, so its PR step would fail where `make_pr`'s
succeeded. One source of truth, two entry points.
