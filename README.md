# git-webui

A small `aiohttp` app for applying patches to Git repositories from a browser UI.

`git-webui` keeps a local clone cache, lets you choose branch workflow modes, applies a patch with `git apply --3way -v`, and can optionally commit and push.

## Current capabilities

- WebSocket-based backend API (`/ws`) with real-time log streaming.
- Built-in frontend served by default from the backend process.
- Repository cache under `repos/` (configurable) keyed by repository URL.
- Config-driven Git identities and SSH keys (no secret material sent to the browser).
- Branch workflow modes:
  - **default**: checkout/switch a branch (or default branch when omitted)
  - **from_commit**: create and push a new branch from a specific commit (or default branch when `HEAD`)
  - **orphan**: create and push a new orphan branch
  - **revert_to_commit**: hard-reset an existing branch to a commit and force-push
- Patch application from pasted text (not file upload) using `git apply --3way -v`.
- Optional empty commit mode.
- Commit message helpers in the UI (clipboard paste/copy prompt and ChatGPT link).

## Requirements

- Python 3.10+
- Dependencies in `requirements.txt`

Install:

```bash
pip install -r requirements.txt
```

## Running

Start backend + frontend (default behavior):

```bash
python backend/app.py
```

Disable frontend hosting (use your own static server):

```bash
python backend/app.py --no-serve-frontend
```

Useful CLI options:

- `--bind` (default: `0.0.0.0`)
- `--port` (default: `8080`)
- `--config` (default: `config.toml`)
- `--repo-root` (default: `repos`)
- `--keep-temp` (keep temporary workspaces for debugging)

## Configuration file

The app reads `config.toml` (or the file given via `--config`).
Use `config-sample.toml` as the reference template when creating your local `config.toml`.

Notes:

- `ssh_keys[].path` is used only on the backend to build `GIT_SSH_COMMAND`.
- Browser clients receive only safe metadata (`label`, default flags, indices).
- `git_users[].default_repositories` is used by the frontend to auto-suggest/select a user by **case-insensitive partial substring matching** against the normalized repository name (last path segment after trimming protocol/host and removing `.git`).

## Frontend behavior

- The backend URL field expects an absolute WebSocket endpoint (examples: `http://localhost:8080/ws`, `ws://localhost:8080/ws`).
- Recent form values and backend URL are stored in browser local storage.
- Query-string prefilling is supported for:
  - `repository_url`, `branch`, `branch_mode`, `new_branch`, `base_commit`
  - `git_user`, `commit_message`, `pr_message`
  - `allow_empty_commit`
  - `ssh_key_path`
  - `patch`
- If query params are present, unspecified fields use defaults rather than previous draft values.

## Backend protocol (WebSocket)

Client -> server message types:

- `{"type":"health"}`
- `{"type":"config"}`
- `{"type":"submit","payload":{...form fields...}}`

Server -> client message types:

- `{"type":"health","status":"ok"}`
- `{"type":"config","payload":{...}}`
- `{"type":"log","line":"..."}`
- `{"type":"complete","success":true|false}`
- `{"type":"error","message":"..."}`

## Operational notes

- Cached repositories are fetched/reset before normal patch workflows to reduce stale state issues.
- Git commands are executed with hooks disabled (`core.hooksPath` to null device).
- Logs include UTC timestamps and include command output plus exit codes.
- If no commit message is provided, patch application can still run but commit/push is skipped.
