---
name: commit-push
description: The write counterpart to review-detailed (which is read-only). Applies terraform fmt (writing, not just -check), runs yamllint/tfsec as a final gate, creates a feature branch if currently on main/master, stages and commits the changes, scans for leaked secrets before pushing, pushes after explicit confirmation, and opens the Azure DevOps PR (always targeting main) using az repos pr create. Use when the user wants to actually commit, push, and PR a change (e.g. after a review or a handoff agent's fixes), not just review it.
---

# Commit & Push

Finishes what `review-detailed` deliberately doesn't: actually applies formatting, commits, pushes, and opens the PR — with a branch-safety check and a mandatory secret scan gate before anything reaches the remote. Local, reversible steps (fmt, branch, commit) proceed without asking; the remote-visible steps (push, PR creation) always get an explicit confirmation, per this repo's git safety rules. This is Azure DevOps (not GitHub) — PR creation uses `az repos pr create`, and the target branch is always `main`.

## Step 0 — Check for uncommitted risk before touching anything

Run `git status` first, always. If there's unrelated in-progress work mixed in with what you're about to commit, stop and ask which files belong to this commit rather than guessing — never stage things you can't account for.

## Step 1 — Apply formatting (write mode)

Unlike `review-detailed`'s `terraform fmt -check`, this step actually rewrites files:

```bash
terraform fmt -recursive
```

Run it from the repo root (or each root module if formatting is scoped) so every touched `.tf` file is normalized. Show the user a summary of what changed if `fmt` rewrote anything beyond what they expected.

## Step 2 — Final lint/security gate

Before staging, run the same checks `review-detailed` uses, but treat any failure as a blocker, not just a report item:

- `yamllint` on any changed `.yml`/`.yaml` files. `yamllint` does not autofix — if it reports errors, stop and ask the user to fix them (or fix them yourself if trivial and the user confirms) before continuing. Do not commit code that fails lint.
- `tfsec <directory> --minimum-severity HIGH` on any changed Terraform directory, if `tfsec` is installed (skip with a note if it isn't — don't install it silently). A HIGH/CRITICAL finding on code this commit is introducing is a blocker: stop and ask the user whether to fix it first or explicitly accept the risk before committing. A finding on pre-existing code that isn't part of this diff is not a blocker — just mention it.
- `terraform validate` in any changed Terraform directory. Treat errors as blockers.

Do not proceed to staging until these are clean or the user has explicitly accepted a specific exception.

## Step 3 — Branch safety

```bash
git branch --show-current
```

If the current branch is `main` or `master`, **never commit directly to it**. Create and switch to a new branch first:

```bash
git checkout -b <branch-name>
```

Derive `<branch-name>` from the nature of the change (e.g. `fix/acr-public-network-access`, `chore/tfsec-findings`) — kebab-case, prefixed with `fix/`, `feat/`, or `chore/` as appropriate. Show the derived name to the user as part of the commit summary in Step 5 rather than asking a separate question, unless the change is ambiguous enough that a wrong guess would be confusing.

If already on a non-main branch, commit there — don't create a new branch unless the user asks for one.

## Step 4 — Stage deliberately

Stage specific files by name — never `git add -A` or `git add .`. Review `git status` after staging to confirm nothing unexpected got included.

Before staging, scan the file list for anything that shouldn't be committed at all: `.env`, `*.tfstate`, `*.tfvars` containing real (non-placeholder) values, `*.pem`/`*.pfx`/`id_rsa`/other key material, `terraform-state-backup-*`, or anything else that looks like a credential or state file rather than source. If found, exclude it from staging and tell the user explicitly — don't silently drop it without saying so, and don't commit it without asking first.

## Step 5 — Commit

Draft a commit message following this repo's existing log style (check `git log` for tone/format), focused on *why*, not just *what*. Never use `--amend` (create a new commit instead), never `--no-verify`, never bypass GPG signing, unless the user explicitly asks. If a pre-commit hook fails, fix the underlying issue and commit again — don't skip the hook.

```bash
git commit -m "$(cat <<'EOF'
<message>

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

## Step 6 — Secret scan before push (mandatory, blocking)

This gate runs regardless of Step 2's tfsec pass — tfsec checks Terraform misconfiguration, not leaked credentials, and this is the last checkpoint before anything leaves the machine.

Scope the scan to what's actually about to be pushed — the commits on this branch not yet on the remote:

```bash
git log --oneline @{u}.. 2>/dev/null || git log --oneline origin/main..HEAD
```

Prefer `gitleaks` if installed:

```bash
gitleaks detect --source . --log-opts="<base>..HEAD" --no-banner
```

If `gitleaks` isn't installed, say so explicitly, then fall back to a manual pass over the diff being pushed (`git diff <base>..HEAD`):
- Look for private key headers (`BEGIN PRIVATE KEY`, `BEGIN RSA PRIVATE KEY`, `BEGIN OPENSSH PRIVATE KEY`).
- Look for assignments like `password =`, `token =`, `secret =`, `api_key =`, `pat_token =`, `connection_string =` followed by a non-placeholder-looking value (not `var.`, not `""`, not obviously an example/dummy).
- Look for AWS-style (`AKIA[0-9A-Z]{16}`), Azure SAS (`sig=`), or other recognizable credential/token shapes.
- Look for any `*.tfstate`, `*.tfvars`, or backup state files that slipped into the commit despite Step 4.

**If anything suspicious is found: stop. Do not push.** Report exactly what was found (file:line) and let the user decide how to handle it — remove and recommit, rewrite history if it was already committed, or confirm it's a false positive (e.g. a placeholder/example value). Never push past a suspected secret on your own judgment.

If the scan is clean, say so explicitly before moving on.

## Step 7 — Push (explicit confirmation required)

Pushing is visible to others and can't be trivially undone — always confirm with the user first, showing: the branch name, the commit(s) about to be pushed, and confirmation that the secret scan was clean. You can fold this into the same confirmation as Step 8's PR creation (both are visible-to-others actions happening back to back) rather than asking twice, as long as the user clearly sees both what's being pushed and what PR will be opened before either happens. Only after confirmation:

```bash
git push -u origin <branch-name>
```

Never force-push. Never push directly to `main`/`master` (Step 3 should already have prevented being on it).

## Step 8 — Open the PR (Azure DevOps, target always `main`)

This org uses Azure DevOps, not GitHub — use `az repos pr create` (the `azure-devops` az CLI extension), never `gh`. The PR always targets `main`; never ask which branch to target.

1. **Check the extension is available**: `az extension show --name azure-devops`. If it's missing, ask before installing (`az extension add --name azure-devops`) — don't add it silently. If `az account show` shows no active login, ask the user to run `! az login` themselves rather than attempting it for them.
2. **Determine org/project/repository** by parsing `git remote get-url origin` — Azure DevOps remotes look like `https://dev.azure.com/<org>/<project>/_git/<repo>` or the legacy `https://<org>.visualstudio.com/<project>/_git/<repo>`. If parsing is ambiguous or fails, ask the user for org/project rather than guessing.
3. **Draft a title and description**: title from the commit message; description summarizing what changed and why. If this branch came from `grill-me`/`execute-task`, pull the Task and Approach sections from that report into the description instead of writing generic filler — the PR reviewer gets the same context the implementing agent had.
4. **Show the drafted title/description to the user** as part of the Step 7 confirmation before creating anything.
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

## Step 9 — Report

Summarize: branch created (if any), commit hash + message, files committed (and any excluded for looking like secrets/state), lint/tfsec/validate results, secret-scan result, push result, and the PR URL/ID.
