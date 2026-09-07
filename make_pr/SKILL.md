---
name: make_pr
description: Ask the user for a branch name, run fmt/lint/validate as a blocking gate, verify .gitignore is sound, then commit and open an Azure DevOps PR. Use when the user wants to wrap up working-tree changes into a branch + PR, e.g. "/make_pr", "make a PR", "ship this as a PR".
---

# Make PR

Interactive, gated commit → PR flow: ask for the branch name up front (never
guess it silently), run every check as a blocker rather than a report item,
explicitly verify `.gitignore` won't let junk/secrets slip in, then commit,
push, and open the Azure DevOps PR (`az repos pr create`, always targeting
`main` — this org is Azure DevOps, not GitHub).

Local, reversible steps (fmt, branch create, commit) proceed without asking.
Remote-visible steps (push, PR creation) always get explicit confirmation.

## Step 0 — Look before touching anything

```bash
git status
git branch --show-current
```

If there's unrelated in-progress work mixed in with what's about to be
committed, stop and ask which files actually belong to this PR — never stage
things you can't account for.

## Step 1 — Ask for the branch name

Use `AskUserQuestion` to get the branch name explicitly — don't derive and
silently apply one. Suggest 2-3 candidates (kebab-case, `fix/`/`feat/`/`chore/`
prefixed, derived from the diff you just saw in Step 0) so the user has a fast
path, but they can always type their own via "Other":

- `<derived-name-1>` — recommended, based on `<one-line reason from the diff>`
- `<derived-name-2>` — alternative framing
- Let them type a custom name via "Other"

If the current branch is already a non-main branch that clearly matches the
same change, offer "Stay on `<current-branch>`" as an option instead of
forcing a new one.

## Step 2 — Apply formatting (write mode)

```bash
terraform fmt -recursive
```

Run from the repo root (or each root module if formatting is scoped there).
Show the user a summary of what changed if `fmt` rewrote anything beyond what
they'd expect.

## Step 3 — Lint / validate gate (blocking)

Treat every failure here as a blocker, not a report item — do not proceed to
staging until these are clean or the user has explicitly accepted a specific,
named exception:

- `yamllint` on any changed `.yml`/`.yaml` files. It doesn't autofix — stop and
  ask the user to fix reported errors (or fix them yourself if trivial and the
  user confirms).
- `terraform validate` in any changed Terraform directory. Errors are blockers.
- `tfsec <directory> --minimum-severity HIGH` on any changed Terraform
  directory, if `tfsec` is installed (skip with a note if it isn't — don't
  install it silently). A HIGH/CRITICAL finding on code this change
  *introduces* is a blocker: ask the user to fix it or explicitly accept the
  risk before continuing. A finding on pre-existing code outside this diff is
  not a blocker, just mention it.

## Step 4 — `.gitignore` sanity check

This is the check that's easy to skip and easy to regret. Before staging:

1. Confirm `.gitignore` exists at the repo root. If it doesn't, stop and ask
   the user whether to create one before continuing.
2. Check it actually covers the categories of file this repo generates, and
   flag anything missing rather than assuming it's fine:
   - `.terraform/`, `.terraform.lock.hcl` (if the repo doesn't intentionally
     commit the lock file — check current tracked files to be sure either way)
   - `*.tfstate`, `*.tfstate.backup`, `terraform.tfstate.*`
   - `*.tfplan`, `*.tfplan.*`, crash logs (`crash.log`)
   - `override.tf`, `override.tf.json`, `*_override.tf`
   - `.env`, `*.pem`, `*.pfx`, `id_rsa*`, other obvious key material
   - Editor/OS cruft already present in the repo (`.DS_Store`, `.vscode/`, etc.)
     — only flag these if you actually see stray instances of them untracked.
3. Cross-check against what's *actually* about to be staged (`git status
   --short`): if anything matching the patterns above shows up as untracked or
   modified and would be swept in, that's a real problem right now, not a
   hypothetical — surface it immediately, and don't stage it.
4. If `.gitignore` is missing entries that matter, propose the specific lines
   to add and ask before editing it (editing `.gitignore` is a real, if small,
   content change — don't do it silently).

## Step 5 — Branch safety

If the user picked a new branch name in Step 1:

```bash
git checkout -b <branch-name>
```

If they picked "stay on `<current-branch>`", skip this — just confirm you're
not on `main`/`master` before continuing. Never commit directly to
`main`/`master` under any circumstance; if somehow still on one here, stop and
create the branch from Step 1's answer instead.

## Step 6 — Stage deliberately

Stage specific files by name — never `git add -A` or `git add .`. Review `git
status` after staging to confirm nothing unexpected got included.

Before staging, scan the file list one more time for anything that shouldn't
be committed at all: `.env`, `*.tfstate`, `*.tfvars` containing real
(non-placeholder) values, key material, `terraform-state-backup-*`, or
anything else that looks like a credential or state file rather than source.
If found, exclude it and tell the user explicitly — don't silently drop it,
and don't commit it without asking first.

## Step 7 — Commit

Draft a commit message following this repo's existing log style (`git log`
for tone/format), focused on *why*, not just *what*. Never `--amend` (create a
new commit instead), never `--no-verify`, never bypass GPG signing, unless the
user explicitly asks. If a pre-commit hook fails, fix the underlying issue and
commit again — don't skip the hook.

```bash
git commit -m "$(cat <<'EOF'
<message>

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

## Step 8 — Secret scan before push (mandatory, blocking)

This runs regardless of Step 3's tfsec pass — tfsec checks Terraform
misconfiguration, not leaked credentials, and this is the last checkpoint
before anything leaves the machine. Scope it to what's actually about to be
pushed:

```bash
git log --oneline @{u}.. 2>/dev/null || git log --oneline origin/main..HEAD
```

Prefer `gitleaks` if installed:

```bash
gitleaks detect --source . --log-opts="<base>..HEAD" --no-banner
```

If it isn't installed, say so explicitly, then fall back to a manual pass over
`git diff <base>..HEAD`:
- Private key headers (`BEGIN PRIVATE KEY`, `BEGIN RSA PRIVATE KEY`, `BEGIN
  OPENSSH PRIVATE KEY`).
- Assignments like `password =`, `token =`, `secret =`, `api_key =`,
  `pat_token =`, `connection_string =` followed by a non-placeholder value.
- AWS-style (`AKIA[0-9A-Z]{16}`), Azure SAS (`sig=`), or other recognizable
  credential/token shapes.
- Any `*.tfstate`, `*.tfvars`, or backup state files that slipped past Step 6.

**If anything suspicious turns up: stop, do not push.** Report exactly what
was found (file:line) and let the user decide — remove and recommit, rewrite
history if already committed, or confirm it's a false positive. Never push
past a suspected secret on your own judgment. If clean, say so explicitly.

## Step 9 — Push (explicit confirmation required)

Pushing is visible to others and can't be trivially undone — confirm first,
showing the branch name, the commit(s) about to be pushed, and that the secret
scan was clean. This can be folded into the same confirmation as Step 10's PR
creation (both are visible-to-others actions back to back), as long as the
user clearly sees both what's being pushed and what PR will be opened before
either happens.

```bash
git push -u origin <branch-name>
```

Never force-push. Never push directly to `main`/`master`.

## Step 10 — Open the PR (Azure DevOps, target always `main`)

This org uses Azure DevOps, not GitHub — use `az repos pr create` (the
`azure-devops` az CLI extension), never `gh`. The PR always targets `main`;
never ask which branch to target.

1. **Check the extension is available**: `az extension show --name
   azure-devops`. If missing, ask before installing (`az extension add --name
   azure-devops`) — don't add it silently. If `az account show` shows no
   active login, ask the user to run `! az login` themselves.
2. **Determine org/project/repository** from `git remote get-url origin`
   (`https://dev.azure.com/<org>/<project>/_git/<repo>`, the legacy
   `https://<org>.visualstudio.com/<project>/_git/<repo>`, or the SSH form
   `git@ssh.dev.azure.com:v3/<org>/<project>/<repo>`). If parsing is
   ambiguous, ask the user rather than guessing.
3. **Draft a title and description**: title from the commit message;
   description summarizing what changed and why, plus a short note on what was
   verified (Step 3's checks, Step 4's `.gitignore` result).
4. **Show the drafted title/description** as part of the Step 9 confirmation
   before creating anything.
5. **Create it**, once confirmed:
   ```bash
   az repos pr create \
     --organization "https://dev.azure.com/<org>" \
     --project "<project>" \
     --repository "<repo>" \
     --source-branch "<branch-name>" \
     --target-branch "main" \
     --title "<title>" \
     --description "<description>"
   ```
6. Capture the returned PR URL/ID for the final report.

## Step 11 — Report

Summarize: branch name (created or reused), commit hash + message, files
committed (and any excluded for looking like secrets/state), fmt/lint/tfsec/
validate results, `.gitignore` check result (clean, or what was added),
secret-scan result, push result, and the PR URL/ID.

## Usage as a command

When invoked as `/make_pr`, run the full flow above in order. It is
interactive at Step 1 (branch name), any Step 3/4 exception, and Steps 9/10
(push + PR confirmation) — don't skip those gates even if earlier steps went
smoothly.
