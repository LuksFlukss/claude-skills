# claude-skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills.

## Install

Claude Code loads skills from `~/.claude/skills/<skill-name>/SKILL.md`. After
cloning this repo, get each skill directory into that folder — symlinking
keeps them in sync with future `git pull`s, so prefer it over copying:

```bash
git clone git@github.com:LuksFlukss/claude-skills.git
mkdir -p ~/.claude/skills
for d in claude-skills/*/; do
  ln -s "$(pwd)/$d" ~/.claude/skills/"$(basename "$d")"
done
```

To copy instead of symlink (won't pick up future updates automatically):

```bash
cp -r claude-skills/*/ ~/.claude/skills/
```

Restart Claude Code (or start a new session) so it picks up the new skills.

## Skills

| Skill | What it does |
|-------|---------------|
| `code-review` | Comprehensive code review — dead code, HIGH/MEDIUM vulnerabilities, best practices, itemized findings. |
| `commit-push` | Same gated flow as `make_pr`, but derives the branch name instead of asking. Thin pointer — the steps live in `make_pr`. |
| `debug-error` | Dispatches an error to two parallel Herdr agents for independent root-cause diagnosis (diagnosis only, no fix applied). |
| `kanban-task` | Discovery → solution design → multi-agent Herdr verification → pitch → approval → push to the kanban board. |
| `kanban-task-eurocontrol` | Eurocontrol-environment variant of `kanban-task` — Claude is the only agent kind installed there, single hardcoded board, no repo routing. |
| `make_pr` | Branch + fmt/lint/validate gate + commit + Azure DevOps PR. |
| `product-owner` | Kanban board maintenance: pitch ideas, groom board health, author cards. |
| `setup-kanban-board` | Starts the taskwarrior-kanban board and onboards a project onto it. |
| `termius` | Reconnects Tailscale without it hijacking routes/DNS. |
| `update-readme` | Rewrites a repo's README to a short, consistent template. |
| `worker` | Executes a kanban task via multi-agent Herdr orchestration with human approval gates. |
| `worker-eurocontrol` | Eurocontrol-environment variant of `worker` — Claude-only, single hardcoded board, no repo routing. |

`shared/` isn't a skill (no `SKILL.md`) — it's reference docs the skills link
to via relative paths, so **it must be installed alongside them** (the
install commands above already do this — `shared/` is just another top-level
directory they symlink/copy). Don't delete or rename it without updating
every skill that references it.

| Shared doc | Contents | Referenced by |
|------------|----------|---------------|
| `shared/agent-routing.md` | Which Herdr agent/model per task type + cost tier, OpenCode `build`/`plan` personas, effort selection, billing-failure fallback, and the legal board `agent:` tokens. | `kanban-task`, `worker`, `product-owner`, `debug-error` |
| `shared/card-template.md` | The Taskwarrior card template (mandatory Verification Plan; Rollout & Blast Radius when a change reaches beyond one repo). | `kanban-task`, `product-owner`, `kanban-task-eurocontrol` |
| `shared/herdr-operations.md` | Herdr mechanics: one-new-tab-per-agent, the permission-approval loop, first-prompt stalls, reading output without burning tokens, cleanup. | all Herdr-using skills |

Adding an agent to `shared/agent-routing.md` also requires adding its token
to `uda.agent.values` in `~/.taskrc`, or every card assigned to it fails to
push.

## Prerequisites

Some skills assume tools/services not covered by this repo itself — check the
individual `SKILL.md`'s own "Prerequisites" section before using it:

- **Herdr CLI** + `HERDR_ENV=1` — `debug-error`, `kanban-task`,
  `kanban-task-eurocontrol`, `worker`, `worker-eurocontrol`.
- **Taskwarrior + taskwarrior-kanban** running at `http://127.0.0.1:8787/` —
  `kanban-task`, `kanban-task-eurocontrol`, `product-owner`,
  `setup-kanban-board`, `worker`, `worker-eurocontrol`.
- **Azure CLI** (`az`) — `commit-push`, `make_pr` (Azure DevOps PRs).
- **Tailscale** — `termius`.
- `kanban-task-eurocontrol` / `worker-eurocontrol` assume an environment
  where **only Claude** is installed as a Herdr agent kind (no OpenCode, no
  Grok, no Google Antigravity) — use the plain `kanban-task`/`worker` skills
  everywhere else.
