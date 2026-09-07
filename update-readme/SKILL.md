---
name: update-readme
description: Analyze a repo and (re)write its README.md to a short, consistent template — a brief description of what the code does, not an exhaustive doc. Input is the target repo (defaults to the current working directory). Use when the user asks to update/fix/standardize a README, or wants all repos' READMEs to follow the same style.
---

# Update README

Writes a short, consistent README for a repo — enough for someone new to understand what it does and how to use it, nothing more. Every repo gets the same template (below) so READMEs read the same across all `tf-azure-*` repos rather than each having its own ad-hoc style.

## Step 1 — Determine the target repo

Default to the current working directory. If the user names a different repo/path, use that instead — don't assume when it's ambiguous (e.g. they just say "update the readmes" with no target: ask whether they mean this repo only, or list which ones).

## Step 2 — Read and understand the repo

Build a real understanding before writing anything:
- **Purpose**: what this repo deploys/manages/automates — infer from resource definitions, root module structure, script names, not just an existing README's claims (an existing README may be stale or wrong).
- **Structure**: key directories/files that matter to a newcomer (root modules, shared modules, environments/workspaces, scripts) — skip anything generated (`.terraform/`, lock files) or purely internal.
- **Usage**: the actual commands used to work with this repo — check for existing CI config (`.ci/azure-pipelines.yml`), scripts, or backend/provider config files to get real commands, not guessed ones.
- **Requirements**: only what's non-obvious or would block someone from starting (specific Terraform/provider versions from `.terraform.lock.hcl`/`versions.tf`, required CLI tools, required auth like `az login`).

If there's an existing README, read it too — carry forward anything factually important that your own analysis can't recover (e.g. a non-obvious operational warning, a link to an external doc), but don't preserve its structure or length. Flag for the user anything in the old README that looks materially important but that you're about to drop, rather than silently discarding it.

## Step 3 — Draft the README using the template

Fill in this exact template — keep it short. If a section would be empty or trivial, omit it entirely rather than padding it:

```markdown
# <Repo Name>

<1–3 sentence description: what this repo deploys/manages/automates, and why it exists.>

## What it does

- <bullet — a concrete thing this repo does/manages>
- <bullet>
- <bullet>
(3–6 bullets max — the highest-level things only, not every resource)

## Structure

- `<path>` — <one-line purpose>
- `<path>` — <one-line purpose>
(only non-obvious paths worth calling out — omit this section entirely for a flat/simple repo)

## Usage

```bash
<the real commands used to init/plan/run this repo>
```

## Requirements

- <only genuinely required tools/versions/access — omit this section if there's nothing non-obvious>
```

Target length: short enough to read in under a minute — if the draft is pushing past ~40 lines, cut detail rather than the format.

## Step 4 — Confirm before writing

Show the drafted README to the user before overwriting the file, since this replaces existing content (possibly written by someone else) — even though it's a local, reversible git-tracked change, a README can carry context you can't fully recover once replaced. Call out explicitly anything you deliberately dropped from an existing README per Step 2.

Once confirmed, write it:
- If `README.md` doesn't exist, create it.
- If it exists, overwrite it entirely with the new draft (don't merge/append — the whole point is one consistent style).

## Step 5 — Report

State what repo was updated and, briefly, what changed (new file vs. rewritten, and anything notable dropped from the old version).
