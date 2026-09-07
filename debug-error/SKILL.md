---
name: debug-error
description: Take an error/stack trace/failure the user gives, dispatch it to two independent Herdr agents (typically different kinds, e.g. a Claude session and an OpenCode session) to investigate root cause in parallel, then read both reports and decide what the actual issue is. Diagnosis only — never applies a fix itself. Use when the user says something like "debug this error", "figure out why this is failing", or wants a second opinion on a root cause. Requires HERDR_ENV=1.
---

# Debug Error

Takes an error the user is stuck on and gets two independent agents investigating it in parallel — different kinds when possible, since a second interpreter (not just a second instance) is more likely to catch something the first one's blind spots miss. This skill only diagnoses; it never applies a fix. That's a separate step afterward (`grill-me` for a design pass, or just asking directly to fix it).

## Step 1 — Confirm HERDR_ENV

```bash
test "${HERDR_ENV:-}" = 1
```

If this fails, tell the user this skill requires running inside Herdr and stop. Do not attempt to control agents from outside Herdr.

## Step 2 — Get the error and its context

Ask the user for:
- The exact error/stack trace/failure output (paste it — don't paraphrase from memory).
- Where it happened: which repo/directory, which command was run, which environment/workspace if relevant.
- Anything already tried or ruled out, so both investigating agents don't waste time re-treading it.

If any of this is missing and the error is ambiguous without it (e.g. no idea which repo/command), ask before dispatching — a vague brief just gets you two vague investigations.

## Step 3 — Pick the two investigating agents

Ask the user (`AskUserQuestion`) which two agent kinds to use. Run `herdr agent list` first to show what's idle/available, and `herdr agent` for the supported kinds. Default suggestion: two *different* kinds (e.g. `claude` + `opencode`) rather than two instances of the same kind — different tools have different blind spots, so disagreement between them is more informative than agreement between two copies of the same one. Let the user override this if they'd rather use two of the same kind or specific existing live agents.

**Whichever two kinds are picked, explicitly tell the user which ones (e.g. "dispatching Claude and Grok") before starting them** — don't fold agent selection into the dispatch silently. If either one later turns out to be out of credits/rate-limited (Step 5) — this includes an `opencode`-kind pick (e.g. `-m opencode/big-pickle`) hitting its own quota, not just Grok — don't auto-retry: tell the user and offer **Google Antigravity (`agy` kind)** as the designated fallback via `AskUserQuestion`, labeled `Google Antigravity — fallback for <failed agent> (recommended)`, alongside re-picking any other kind.

Follow the `herdr` skill's conventions for starting new agents: check pane availability with `herdr pane layout`, split sibling panes with `herdr pane split --current --direction right --cwd "$PWD" --no-focus`, then start each agent — the model/agent/effort are the underlying binary's own CLI args, passed after `--`, not `herdr`-level flags:

```bash
herdr agent start <name> --kind claude --pane <pane-id> -- --effort <level>
herdr agent start <name> --kind grok --pane <pane-id> -- --effort <level>
herdr agent start <name> --kind opencode --pane <pane-id> -- -m opencode/<model> --agent <build|plan>
herdr agent start <name> --kind agy --pane <pane-id> -- --model <model-name> --effort <level>
```

**Effort (Claude/Grok/Antigravity only — OpenCode has no effort concept):** before starting a Claude, Grok, or Antigravity investigator, propose a recommended `--effort` level for the diagnosis (e.g. `medium` for a routine error, `high` for one that's resisted a first look or spans multiple subsystems) and confirm with the user via `AskUserQuestion` before spawning. Claude accepts `low`/`medium`/`high`/`xhigh`/`max`; Grok accepts `--reasoning-effort`/`--effort` with `low`/`medium`/`high` confirmed in practice (no tier above `high`); **Google Antigravity** (`agy` — Grok's designated fallback, see Step 3) accepts `--effort <level>` with the same `low`/`medium`/`high` scale and also needs `--model <name>` (run `agy models` to list current options; prefer a `-pro-` tier).

**OpenCode model/agent (if `opencode` is picked):** always pass both `-m <provider/model>` (full form, e.g. `opencode/big-pickle`, `opencode/mimo-v2.5-free`) and `--agent <build|plan>` — `build` for an investigation that needs to run commands/read broadly, `plan` for pure reasoning over what's already been pasted. Ask the user which model if they picked `opencode` without specifying one.

## Step 4 — Compose one shared investigation prompt

Write a single self-contained prompt (both agents get the identical brief, so their outputs are comparable) containing:
1. The exact error/stack trace/output from Step 2.
2. Context: repo, directory, command, environment.
3. What's already been ruled out, if anything.
4. Explicit instructions: investigate root cause only — read code, logs, config, run read-only diagnostic commands (e.g. `terraform validate`, `terraform plan`, test commands) as needed to confirm a hypothesis, but **do not apply any fix or change any file**.
5. A request for a structured reply: **most likely root cause** (with evidence — file:line, log excerpt, command output), **confidence** (high/medium/low), and **suggested fix** (described, not applied).

## Step 5 — Dispatch to both in parallel

Send to both without waiting on either individually, so they run concurrently:

```bash
herdr agent prompt <agentA> "<investigation prompt>" --timeout 300000
herdr agent prompt <agentB> "<investigation prompt>" --timeout 300000
```

Then poll both until each is done rather than blocking on one at a time:

```bash
herdr agent get <agentA>
herdr agent get <agentB>
```

Once each shows idle/complete, read its output:

```bash
herdr agent read <agentA> --source recent-unwrapped --lines 300
herdr agent read <agentB> --source recent-unwrapped --lines 300
```

If either ends up `blocked` (approval/question), inspect with `herdr agent get`/`herdr agent read` and ask the user how to respond rather than answering on their behalf.

## Step 6 — Read both reports and decide

Compare the two independent diagnoses yourself — don't just paste both back and ask the user to pick:
- **Agreement** — both point to the same root cause: highest confidence, lead with this.
- **Disagreement** — surface both hypotheses explicitly, and sanity-check each against the actual code/error yourself (read the file, re-check the log) rather than passing through an unverified claim. State which one you find more convincing and why, or say plainly if you can't tell from here.
- **One found nothing / gave up** — note that, and rely on the other's finding, but flag that only one agent actually converged.

Present one final diagnosis: the most likely root cause, the evidence for it, and the suggested fix — framed as ready input for `grill-me` (if it needs a design pass) or direct implementation, but don't apply anything yourself.
