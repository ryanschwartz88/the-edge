---
name: edge-remote
description: >
  Convert a local Edge setup to cloud-scheduled. Creates a private GitHub repo
  from the user's ~/.the-edge/ contents, pushes it, wires up a daily trigger
  for the-edge and a weekly trigger for edge-refine via the schedule /
  RemoteTrigger system, and sends a test digest from the cloud. After this,
  the laptop can be closed and the digest still arrives. Use when the user
  says "edge-remote", "move the edge to the cloud", "set up cloud triggers",
  or wants laptop-closed delivery.
allowed-tools: Bash Read Write Edit Glob Grep mcp__claude_ai_Gmail__* RemoteTrigger CronCreate CronList
---

# Edge Remote — Promote Local Setup to Cloud-Scheduled

Take an existing `~/.the-edge/` setup and put it behind a cloud-scheduled trigger so the daily brief arrives even when the laptop is closed.

This is an **opt-in** step. Users who are fine with local `launchd` scheduling can skip this skill entirely.

## What this skill produces

1. A **private** GitHub repo on the user's account (default name `the-edge-state`).
2. `~/.the-edge/` converted into a git checkout of that repo — same files, now tracked and pushed.
3. A **daily** remote trigger that runs `the-edge` at the user's send time.
4. A **weekly** remote trigger that runs `edge-refine` on Sundays.
5. A test run from the cloud to confirm end-to-end delivery.

## Prerequisites

Before starting, verify:

- `~/.the-edge/config/sources.md` exists. If not, tell the user to run `edge-initialize` first and stop.
- `gh` CLI installed and authenticated. If not: instruct them to run `! gh auth login` in the Claude prompt, then resume.
- `git` is configured with `user.name` and `user.email`. If not: surface and stop.
- The `schedule`/`RemoteTrigger` system is available (check for `RemoteTrigger` / `CronCreate` tools). If not, surface and stop.
- `~/.the-edge/.git` does not already exist (meaning this hasn't been run before). If it does, ask whether to re-register triggers only, or refuse.

If any prereq fails, stop and surface exactly what's needed.

## Interview

Ask:

1. **Repo name** — default `the-edge-state`. Confirm no collision with an existing repo on the user's account via `gh repo view {user}/{name}`.
2. **Repo visibility** — default `private`. Public is allowed but surface a warning that `data/digests/` will be public.

## Pipeline

### 1. Initialize git in `~/.the-edge/`

```bash
cd ~/.the-edge
git init -b main
git add .
git commit -m "init: personal edge state"
```

### 2. Create the remote repo and push

```bash
gh repo create {user}/{name} --{private|public} --source=. --remote=origin --push
```

Confirm the repo is visible on GitHub before continuing.

### 3. Parse send time from config

Read `~/.the-edge/config/sources.md` frontmatter. Extract `timezone` and `delivery.send_time`. Convert to UTC for the cron expression.

Example: `America/Los_Angeles` + `07:00` → `0 14 * * *` UTC (adjust for DST by using the trigger's timezone field if supported, otherwise compute for current offset and document it).

### 4. Create the daily trigger

Use `RemoteTrigger` / `CronCreate` to register:

- **Schedule**: the cron above.
- **Target repo**: `{user}/{name}`, branch `main`.
- **Prompt**:
  ```
  Run the the-edge skill with default flags against the current checkout.
  State is in this repo (config/sources.md, data/digests/). Commit and push
  the resulting digest to main.
  ```
- **Permissions**: read/write to the target repo, Gmail MCP, web fetch.

Save the trigger ID returned.

### 5. Create the weekly trigger

- **Schedule**: Sunday at send_time + 2h UTC (gives room before the next daily).
- **Target repo**: same.
- **Prompt**:
  ```
  Run the edge-refine skill against the current checkout. Open a PR with
  proposed source changes and email the summary.
  ```

Save the trigger ID.

### 6. Fire a one-off test run

Invoke the daily trigger manually (most trigger systems expose a "run now" option — use it). Wait for completion. Ask the user to confirm:
- The test email arrived.
- A new commit appeared on `main` of the repo.

If either fails, surface the error and stop before marking the skill complete.

### 7. Summary to user

Report:
- Repo URL
- Daily trigger ID + next fire time (in user's timezone)
- Weekly trigger ID + next fire time
- Management: `/schedule list`, `/schedule delete <id>`
- How to edit config going forward: edit `~/.the-edge/config/sources.md` locally, then `git push` (or edit directly on GitHub). The cloud trigger pulls fresh each run.
- If they want to pause: `/schedule delete <daily_id>` — local files remain intact.

## Guardrails

- **Private by default.** Only create public repos if the user explicitly chose that in the interview.
- **Never force-push**, ever.
- **Never commit secrets** — double-check `.gitignore` excludes `.env`, `*.key`, `.ssh/`, etc. Write a `.gitignore` if one doesn't exist in `~/.the-edge/`.
- If `~/.the-edge/` already has a `.git` directory, do not re-init — refuse and ask the user to delete it first or use a different path.
- If any prerequisite tool is missing, stop before the first destructive step. Don't half-configure.

## Reversal

If the user wants to go back to local-only:

1. `/schedule delete <daily_id>` and `/schedule delete <weekly_id>`.
2. Optional: `gh repo delete {user}/{name}` to remove the remote.
3. Optional: `cd ~/.the-edge && rm -rf .git` to remove git tracking, leaving the files as plain local state.

Document these steps in the summary so the user knows how to undo.
