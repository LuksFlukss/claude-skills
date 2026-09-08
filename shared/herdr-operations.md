# Shared: Herdr Operational Conventions

How to actually drive Herdr agents without burning tokens or stalling.
Referenced by `kanban-task`, `worker`, `product-owner`, `debug-error`, and the
`-eurocontrol` variants. Every item here was hit in practice — this file
exists so each run stops rediscovering the same friction.

For *which* agent/model to pick, see
[`agent-routing.md`](agent-routing.md). This file is only about mechanics.

## 1. One agent = one brand-new tab

**Every agent gets its own new tab. Never a `herdr pane split` off the
orchestrator's tab, never a reused idle pane.** A split shares screen space
with the orchestrator's own view and gets cramped as agents pile in; an
idle-looking pane may belong to the user's own session or another task's
agent, so reusing it risks clobbering work-in-progress or mixing two agents'
context together.

```bash
TAB_JSON=$(herdr tab create --cwd "$PWD" --no-focus)   # --cwd = the target repo
TAB_ID=$(echo "$TAB_JSON"  | jq -r '.result.tab.tab_id')
PANE_ID=$(echo "$TAB_JSON" | jq -r '.result.root_pane.pane_id')
herdr agent start <name> --kind <kind> --pane "$PANE_ID" -- <binary's own args>
```

Track `name → {tab_id, pane_id, kind, model}` for every agent started, and
close the **whole tab** when done (§6) — not just the pane.

Anything the agent's binary needs (`--effort`, `-m`, `--agent`, `--model`)
goes **after `--`**; those are the underlying CLI's flags, not Herdr's.

## 2. Blocked on a permission prompt — the approval loop

Agents reading anything **outside their `--cwd`** (a sibling repo, `~/`) hit
an interactive "Access external directory" prompt and go `agent_status:
blocked`. It fires **once per directory**, so a cross-repo scan can block
five or more times in a row. Nothing proceeds until each is answered.

`Allow once` is the default-focused choice, so a bare `enter` accepts it:

```bash
for i in $(seq 1 30); do
  S=$(herdr agent get <name> 2>&1 | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['agent']['agent_status'])" 2>/dev/null)
  echo "poll $i: $S"
  case "$S" in
    blocked)    herdr agent send-keys <name> enter >/dev/null 2>&1; sleep 4; continue ;;
    done|idle)  break ;;
  esac
  sleep 8
done
```

Two rules:
- **Only auto-accept read-only access for an agent you launched read-only**
  (`--agent plan`, or a brief that forbids writes). Anything else — a write,
  a destructive command, a real question — goes to the **user**; never answer
  an approval on their behalf.
- If a prompt keeps reappearing for the same path, the agent is looping;
  stop it rather than approving indefinitely.

Prefer avoiding the loop entirely: set `--cwd` to the directory the agent
actually needs, and when a brief genuinely spans repos, say so in the prompt
so the agent front-loads those reads instead of trickling them out.

## 3. First prompt to a fresh agent can stall — retry once

`herdr agent prompt` immediately after `herdr agent start` can come back with
`agent_prompt_stalled` ("no observed state change within 5000 ms") while the
agent's TUI is still coming up. The agent is fine. Confirm it's alive, then
send the same prompt again:

```bash
herdr agent get <name>                 # expect agent_status: idle
herdr agent prompt <name> "..." --wait --timeout 600000
```

Seen consistently with `agy`. Retry once before treating it as a real
failure — and don't confuse it with a billing/quota error, which
`agent-routing.md`'s fallback section handles differently.

Likewise, `--wait` returning a timeout is **not** proof of failure: a
substantial proposal routinely outlives the timeout. Poll `herdr agent get`
for `done`/`idle` before concluding anything.

## 4. Reading agent output without burning tokens

**This is the single biggest token sink in these workflows.** `herdr agent
read` scrapes the pane, so you get the TUI as-rendered: box-drawing borders,
a per-line status gutter (`Context 27,460 tokens / 14% used / $0.00 spent`),
and lines duplicated by redraws. Roughly **40-50% of a raw transcript is
chrome**, and it lands in the orchestrator's context verbatim.

Do this instead, in order of preference:

1. **Ask for a compact report up front.** Put it in the brief: "report back
   in plain text, under N words, no preamble." A tight answer is cheap to
   read; a long one costs twice (once as transcript, once as your summary).
2. **Read conservatively — start at `--lines 150-300`**, and only go wider if
   the answer is visibly truncated. Do **not** default to `--lines 3000`:
   that reliably overflows the tool-output cap, gets spilled to a file, and
   then has to be paged back in chunks — strictly worse than two targeted
   reads.
3. **For `build`-persona agents, have them write the report to a file** and
   read the file directly — no chrome at all:
   ```
   ...write your findings to /tmp/<name>-report.md and reply only "done".
   ```
   (Not available to `--agent plan` agents: they can't write.)
4. **If a long transcript is unavoidable**, ask the agent to restate its
   conclusion in a short follow-up prompt rather than scraping scrollback.

Source selection matters too:
- `--source recent-unwrapped` — full history, **but only while the agent is
  idle**. Called mid-run it errors with `agent_not_idle` ("alternate-screen
  history can only be captured by scrolling while idle").
- `--source visible` — just the current viewport; works **while the agent is
  still working**. Use it to check progress or see what an agent is blocked
  on, not to collect a final report.

## 5. Reconciling several agents

When agents disagree on a fact (a live test count, whether a file exists),
**settle it yourself by re-running the check** — don't average the reports or
pick one at random. When they disagree on a judgment call, say so explicitly
rather than silently adopting one.

An agent that reports nothing usable (stalled, wandered off-brief, or ended
mid-tool-call) counts as **no data**, not as agreement. Say so, and don't
launder its silence into consensus.

## 6. Cleanup

Close every tab you opened, once its work is captured and the task is
finished:

```bash
herdr agent list     # confirm nothing you started is still needed
herdr tab close <tab_id>
```

Close the **tab**, not the pane — everything you started came from `herdr tab
create` (§1), so nothing there is the user's own pane. Idle agents sometimes
exit and drop their tab on their own; a `tab_not_found` on close just means
that already happened, and is not an error worth chasing.
