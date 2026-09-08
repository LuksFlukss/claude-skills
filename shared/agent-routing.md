# Shared: Agent/Model Routing (Herdr Kinds)

Canonical reference for picking a Herdr agent kind/model by task type and
cost tier. Referenced by `kanban-task`, `worker`, and `product-owner` —
**edit here once, not in three places.** This file was extracted after
Nemotron had to be removed from four separate copies of this table in one
sitting; keeping one source of truth avoids that drift happening again.

Each calling skill still owns its own persona-forcing rules (e.g. "this
phase is proposal-only, so force `plan` even on a `build` row") and its own
phase numbering — this file only covers the parts that are identical across
all three: the table itself, the OpenCode invocation shape, effort
selection, and the billing-failure fallback pattern.

## Routing Table

| Task Type | Best-fit Agent | Herdr Kind / Model / OpenCode Agent | Cost Tier | Rationale |
|-----------|---------------|---------------------|-----------|-----------|
| Architecture / high-level design, trade-off analysis | MiMo V2.5 | `opencode` → `-m opencode/mimo-v2.5-free --agent plan` | Free | Strong reasoning/planning at no cost — use for design-heavy tasks. `--agent plan` is OpenCode's read-only design/reasoning persona, matching this task type. |
| Complex multi-file implementation, large refactors | Claude | `claude` (Sonnet/Opus) | $$ (paid) | Best correctness on architecture-sensitive or engine-purity-sensitive changes. |
| Everyday multi-file coding, focused bug fixes, test writing | Big Pickle | `opencode` → `-m opencode/big-pickle --agent build` | $ (low) | Good default for routine implementation, cheaper than Claude. `--agent build` is OpenCode's full-tool-access implementation persona. |
| Long doc/spec reading, quick lint/summarization | Big Pickle | `opencode` → `-m opencode/big-pickle --agent build` | $ (low) | Same agent as the everyday-coding row — Nemotron is out of rotation, so Big Pickle also covers read-heavy or mechanical sub-tasks. |
| Alternative cross-check / diverse second opinion, broad-context reasoning | Grok | `grok` | $ (paid) | Different model family from the rest of the table — good diversity pick for a verification cross-check when both other slots would otherwise be opencode-based. |
| Fallback (used only when Grok, or any OpenCode-kind agent above — MiMo/Big Pickle — hits its usage/rate limit or runs out of tokens; never a first pick) | Google Antigravity | `agy` → `--model <name> --effort <level>` (run `agy models` first to confirm current model names — prefer a `-pro-` tier for cross-check work, e.g. `gemini-3.1-pro-high`) | Google-account quota | Different model family from every other row — substituted in only when the agent it's replacing is confirmed unavailable. |

**Nemotron (any variant, e.g. `opencode/nemotron-3-ultra-free`) is never
used.** Big Pickle covers both the everyday-coding and long-doc/lint rows
above — do not reach for a Nemotron model even as a one-off substitute.

### Board `agent:` tokens (not the same as the labels above)

When assigning a card (`backlog add --agent <token>` / `backlog assign`), the
Taskwarrior `agent` UDA is an **enum** — anything outside it is rejected
outright, so a display label like `"Big Pickle"` fails. Map label → token:

| Label in this table | `agent:` token |
|---------------------|----------------|
| Claude | `claude` |
| Big Pickle | `big-pickle` |
| MiMo V2.5 | `mimo` |
| Grok | `grok` |
| Google Antigravity | `antigravity` |
| (a human will do it) | `human` |
| (this orchestrator) | `orchestrator` |

Also legal but not routed from this table: `codex`, `mock`, `self_test`, and
`nemotron` (retired — never assign it; it stays legal only until the last
card using it is reassigned). The enum lives in `~/.taskrc`
(`uda.agent.values`, in the `taskwarrior-kanban managed UDAs` block, which is
the definition that actually takes effect). **Adding a new agent to this
table means adding its token there too**, or every push assigning it fails.

**Whenever ANY agent from this table is dispatched, explicitly tell the user
which one (and briefly why) before starting it.** Never fold an agent
selection into a pane-split/start sequence silently.

**Cost discipline:** prefer the Free tier (MiMo) whenever the task type is
architecture/design rather than defaulting to Claude. Only route to Claude
when the task is genuinely complex multi-file work, touches engine-purity
rules, or is itself "architecture" at high risk.

## OpenCode Invocation Shape

Always launch OpenCode-kind agents with both `-m <provider/model>` (the
model, full `provider/model` form — bare model names are not guaranteed to
resolve) **and** `--agent <name>` (OpenCode's own agent persona, separate
from the model choice). Only `build` and `plan` are launchable top-level
personas for our purposes (`explore`/`general` are subagents OpenCode
dispatches internally, not directly startable via this flag):

- `build` — full tool access (reads/writes files, runs commands). Use for
  any sub-task that needs to actually implement/fix/test.
- `plan` — read-only design/reasoning, no file changes. Use for pure
  analysis/design, or whenever the *calling skill's* context is
  proposal-only/diagnosis-only regardless of what this table's row says for
  that agent — each calling skill states its own persona-forcing rule for
  its own context (e.g. "this phase never writes files, so force `plan`
  even on Big Pickle's `build` row").

Via Herdr: `herdr agent start <name> --kind opencode --pane <id> -- -m
opencode/<model> --agent <build|plan>`.

## Effort Selection (Claude, Grok & Antigravity only — OpenCode has no effort concept)

Before starting any Claude, Grok, or Antigravity agent, propose a
recommended effort level based on the sub-task's actual complexity — don't
just default to the tool's default effort — then confirm with the user via
`AskUserQuestion` before spinning the agent up (one question per such agent,
batched into as few calls as fits the 4-question limit). Present the
recommended level first, labeled `<level> — recommended (<one-line
reason>)`, with 1-2 adjacent levels as alternatives.

- **Claude** accepts `--effort <level>`: `low`, `medium`, `high`, `xhigh`,
  `max` (confirmed via `claude --help`). Recommend `medium` for routine,
  well-scoped work; `high` for genuinely complex/architecture-risk work;
  `xhigh`/`max` only for something flagged as unusually high-risk (e.g.
  security/trust-boundary or engine-purity-critical).
- **Grok** accepts `--reasoning-effort <level>` (alias `--effort`);
  OpenCode's CLI help does not enumerate exact accepted values, but
  `low`/`medium`/`high` are confirmed to work in practice (a pane's status
  line showed `Grok 4.6 (high)` after using `high`). Map the same way as
  Claude's scale, with `high` standing in for Claude's `xhigh`/`max` (Grok
  has no tier above `high`).
- **Google Antigravity** (`agy`) also accepts `--effort <level>`:
  `low`/`medium`/`high` (confirmed via `agy --help`) — same three-tier
  scale as Grok, no `xhigh`/`max`. Also needs `--model <name>` (run `agy
  models` to list current options; prefer a `-pro-` tier for cross-check
  work). Note: the `-pro-` model names already encode their own tier (e.g.
  `gemini-3.1-pro-high`), so passing a separate `--effort` alongside one of
  those can conflict — the CLI resolves this by keeping the model's own
  tier and warning, so it's safe to pass both, just don't expect the
  `--effort` value to override a `-pro-` model's baked-in tier.
- Launch with the chosen level: `herdr agent start <name> --kind claude
  --pane <id> -- --effort <level>` / `herdr agent start <name> --kind grok
  --pane <id> -- --effort <level>` / `herdr agent start <name> --kind agy
  --pane <id> -- --model <model-name> --effort <level>`.

## Billing-Failure / Rate-Limit Fallback

If an agent is out of credits / billing-failed (pane output or `herdr agent
read` shows a billing/quota error, or the call errors/times out with no
real response): **do not auto-retry or silently fall back.** Tell the user
which agent/kind failed.

- **If the failed agent is Grok, or an OpenCode-kind agent (Big Pickle,
  MiMo)** — hit its usage/rate limit, or burned through its available
  tokens/quota: say so explicitly, then use `AskUserQuestion` with
  **`Google Antigravity — fallback for <failed agent> (recommended)`**
  (`agy` kind) as the first option, alongside the usual next-best
  alternative(s) for that task type and a "skip this slot" option. This is
  a designated, named fallback — not a generic "pick anything" choice —
  but it is still never applied silently; the user confirms it via the
  question like any other re-dispatch.
- **If the failed agent is Claude**, there is no designated single fallback
  (Claude has no free/quota-limited sibling in this table) — offer the
  next-best alternative(s) instead.
