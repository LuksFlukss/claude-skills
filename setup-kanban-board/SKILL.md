---
name: setup-kanban-board
description: Idempotently start the taskwarrior-kanban board at http://127.0.0.1:8787/ in production mode, and wire the calling project into it (CLAUDE.md + a scoped backlog wrapper). Kills stale dev servers, rebuilds web bundle, verifies startup. Use when you need the kanban board running, or need a project onboarded onto it.
---

# Setup / Start the Kanban Board

Bring up the live Taskwarrior kanban web board (taskwarrior-kanban) at
`http://127.0.0.1:8787/`. It renders the `~/.task` Taskwarrior backlog — **all**
projects share this one board, distinguished by Taskwarrior's `project:` field
(e.g. `guiñotazo`, or whatever the calling project is scoped to) — as a kanban
across `todo → active → review → done`.

This is a single shared server, not one board per project. Running this skill
from a different project directory does not create a second board — it starts
(or reuses) the same `~/.local/share/taskwarrior-kanban` instance, and then
onboards *that* project onto it by scoping its own tasks under
`project:<project-name>` in the shared store.

This skill is idempotent — safe to run any time. If the board is already running,
it reports the URL and exits (still checking/updating the project's CLAUDE.md
in step 5). If a stale dev-mode server holds port 8787, it kills it and starts
fresh in production mode.

## Setup facts (do not rediscover)

- Install dir: `~/.local/share/taskwarrior-kanban`
- Server: Fastify, binds `127.0.0.1:8787` (see `server/src/config.ts`)
- **MUST use production mode**: `npm run start` in the install dir (which runs
  `NODE_ENV=production pnpm -C server start`). The production server serves the
  built web bundle from `web/dist/` AND the API on 8787.
- **DO NOT use `pnpm dev`**: dev mode splits the app — the Fastify API stays on
  8787 but Vite serves the UI on its own port (e.g. 5174 when 5173 is taken by
  the guiñote Vite app), leaving `http://127.0.0.1:8787/` on 404.
- Backend data: Taskwarrior 3.5 taskchampion sqlite store at `~/.task` (shared
  with the `scripts/backlog` wrapper)
- It only reads Taskwarrior; it does not need Postgres or the guiñote Vite app
  to be running.
- The CLI wrapper lives at `~/.claude/skills/scripts/backlog`. It is project-agnostic:
  it scopes every `task` call to `project:${BACKLOG_PROJECT:-guiñotazo}`. Any
  project can use it by exporting `BACKLOG_PROJECT=<that project's name>`
  before calling it — this is how a new project gets its own lane on the
  shared board without forking the script.

## Steps

### 1. Check if it is already up

Curl the board:
```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8787/
```

- If it returns `200` → already running. Report the URL and **stop**.
- If it returns `000` → not running. Proceed to start it.
- If it returns a **non-200 like 404** → a stale dev-mode server is holding the
  port. Kill it first, then proceed:
  ```bash
  kill $(lsof -ti tcp:8787) 2>/dev/null
  ```
  Then confirm `curl` returns `000` before starting.

### 2. Rebuild the web bundle

So the served UI matches the current source:
```bash
cd ~/.local/share/taskwarrior-kanban && pnpm build
```
(Safe to skip if you only want the existing build, but rebuilding avoids
stale-UI confusion.)

### 3. Start in production mode, fully detached

So it survives the shell:
```bash
cd ~/.local/share/taskwarrior-kanban && setsid nohup npm run start > /tmp/taskwarrior-kanban.log 2>&1 < /dev/null & disown
```

### 4. Verify it came up

Wait ~3s, then re-curl `http://127.0.0.1:8787/`.
- If it returns `200` → report the URL.
- If it fails or returns 404 → read `/tmp/taskwarrior-kanban.log` and surface
  the error. `EADDRINUSE` means a stale process still holds 8787 — kill it and
  retry.

### 5. Onboard the calling project onto the board

Run this regardless of whether the board was already up or was just started —
it's about wiring up *this* project, not the server.

1. Derive a project name: the basename of the current working directory
   (e.g. `/home/uu/combu-barata` → `combu-barata`). If the directory is a
   scratch/temp location, ask the user what project name to use instead of
   guessing.
2. Look for `CLAUDE.md` (or `AGENTS.md`) in the project root.
   - **Missing**: create `CLAUDE.md` with a "## Task tracking" section (see
     template below).
   - **Exists, no kanban section**: append a "## Task tracking" section — do
     not touch unrelated content.
   - **Exists, section already present**: leave it alone (idempotent) unless
     the project name or board URL has drifted, then correct it in place.

   Template for the section:
   ```markdown
   ## Task tracking

   This project's work is tracked on the shared taskwarrior-kanban board, not
   as ad-hoc TODOs in chat. Board UI: http://127.0.0.1:8787/ (start it with the
   `setup-kanban-board` skill if it's down).

   Tasks for this project are scoped with `project:<project-name>` in
   Taskwarrior. Use the `backlog` wrapper (`~/.claude/skills/scripts/backlog`) with
   `BACKLOG_PROJECT=<project-name>` exported, e.g.:

   ```bash
   export BACKLOG_PROJECT=<project-name>
   ~/.claude/skills/scripts/backlog add "description" [--agent NAME] [--priority H|M|L]
   ~/.claude/skills/scripts/backlog next     # list open/active tasks
   ~/.claude/skills/scripts/backlog claim    # pick up a task
   ~/.claude/skills/scripts/backlog done     # complete a task
   ```

   When picking up work in this repo, check the board/backlog first instead of
   inventing new tasks silently — file anything non-trivial as a card so it's
   visible across sessions.
   ```

   Replace `<project-name>` with the value derived in step 1.
3. Report which project name was used and whether CLAUDE.md was created,
   appended, or left untouched.

### 6. Report

Confirm `http://127.0.0.1:8787/` serves both UI and API. If it was already
running, say so instead of starting a duplicate. Include the step 5 outcome
(project name, CLAUDE.md action taken).

## Notes

- Idempotent: intended to be safe to invoke any time, e.g. from this session's
  other workflow skills that reference the board.
- Do not start the app/Vite or Postgres unless asked — this skill only manages
  the kanban board. In particular do NOT touch the guiñote Vite app on 5173.
- Step 5 only ever adds/appends a section to CLAUDE.md/AGENTS.md — never
  overwrite or rewrite existing content in that file.

---

## Usage as a command

When invoked as `/setup_kanban_board`, execute the steps above and report the
result.