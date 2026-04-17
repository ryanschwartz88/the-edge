---
name: edge-initialize
description: >
  First-time setup for The Edge. Forks or clones the-edge repo under the user's
  GitHub account, customizes config/sources.md for their topics and email,
  wires up the daily + weekly remote triggers, and sends a test digest. Use
  when the user says "set up the edge", "initialize the edge", "onboard me
  to the edge", or has just installed the skills and has no config yet.
allowed-tools: Bash Read Write Edit Glob Grep WebFetch mcp__claude_ai_Gmail__*
---

# Edge Initialize — First-Time Setup

Turn a fresh install of these skills into a running, personalized daily brief. The user runs this once. After it, `the-edge` runs daily in the cloud and emails them.

## What this skill produces

1. A personal repo on the user's GitHub account (a fork or fresh clone of `ryanschwartz88/the-edge`) that holds their `config/sources.md` and their digest history.
2. A customized `config/sources.md` tailored to their topics, email, and stack.
3. Two remote triggers (daily for `the-edge`, weekly for `edge-refine`) pointed at their repo.
4. A test digest delivered to their inbox as proof it all works.

## Prerequisites — check before starting

Before any step, verify the user has:

- `gh` CLI installed and authenticated (`gh auth status`). If not: instruct them to run `! gh auth login` in the Claude prompt, then continue.
- Gmail MCP available (look for `mcp__claude_ai_Gmail__*` tools). If not: instruct them to authenticate via `mcp__claude_ai_Gmail__authenticate`.
- `git` configured with `user.name` and `user.email`.

If any prereq fails, stop at that step and surface exactly what the user must do. Don't try to work around missing auth.

## Interview

Ask the user these questions, one message per question (don't batch), and record answers:

1. **GitHub username** — where their personal copy of the repo will live.
2. **Email address** — where digests get delivered.
3. **Timezone** — for send-time scheduling.
4. **Send time** — preferred daily delivery time (default 07:00).
5. **Topics** — 3 to 6 themes, specific. Suggest they look at the defaults in the starter `sources.md` and edit.
6. **Stack keywords** — boost-signal keywords tied to what they're building (e.g., `SwiftUI`, `MCP`, `Next.js`).
7. **Existing repo name** — default is `the-edge`. They may pick another name to avoid collisions.

## Pipeline

### 1. Create the personal repo

Prefer a fork so the user keeps an upstream to pull refinements from:

```bash
gh repo fork ryanschwartz88/the-edge --clone=true --fork-name={repo_name}
```

If fork fails or the user wants a clean slate, fall back to a template-style clone:

```bash
git clone https://github.com/ryanschwartz88/the-edge.git {repo_name}
cd {repo_name}
rm -rf .git
git init
gh repo create {user}/{repo_name} --public --source=. --remote=origin --push
```

Confirm the repo is visible on GitHub before continuing.

### 2. Customize `config/sources.md`

In the cloned repo:

- Replace `owner: YOUR_NAME` with the user's GitHub username.
- Replace `email: you@example.com` with the real delivery email.
- Update `timezone` and `delivery.send_time`.
- Replace the `## Topics` list with the user's 3–6 topics.
- Replace the `## Boost signal` placeholders with the user's stack keywords.
- Leave the primary source lists as starter defaults unless the user explicitly wants to edit — `edge-refine` will tune over time.

Commit: `init: personalize sources.md for {user}` and push to origin main.

### 3. Verify with a dry run

From the repo root, invoke the-edge with `--dry-run`:

```
the-edge --dry-run
```

Expected: the skill prints a digest to stdout, sends no email, writes no file. Confirm the output looks reasonable (≥ 5 items, primary sources dominate, hype filter caught at least some items).

If zero items surface, the topic list is probably too narrow — loop back to the interview and widen topics.

### 4. Test email

Invoke `the-edge --no-commit` to send a real email without committing a digest file.

Ask the user to confirm the email arrived. If not: likely a Gmail MCP auth issue. Surface and stop.

### 5. Wire up remote triggers

Use the `schedule` / `RemoteTrigger` system to create two triggers.

**Daily trigger** (`the-edge`):
- Schedule: cron expression matching the user's send time in their timezone. Example for 07:00 America/Los_Angeles: `0 14 * * *` UTC (14:00 UTC = 07:00 PT).
- Target repo: `{user}/{repo_name}` on branch `main`.
- Prompt: `Run the the-edge skill with default flags. Commit and push the resulting digest.`
- Permissions required: read/write to the target repo, Gmail MCP, web fetch.

**Weekly trigger** (`edge-refine`):
- Schedule: Sunday at `send_time + 2h` (gives room before the next daily run). Example: `0 16 * * 0` UTC.
- Target repo: same.
- Prompt: `Run the edge-refine skill. Open a PR with proposed source changes and email the summary.`

Hand the trigger IDs back to the user and note where they manage them (`/schedule list`).

### 6. First real commit

Run `the-edge` for real (no flags) so day-one output is in the repo. Confirm:
- Email arrived.
- `data/digests/YYYY-MM-DD.md` was committed and pushed.
- `git log` on the repo shows the commit.

### 7. Summary to user

Report:
- Repo URL
- Trigger IDs and when they'll next fire
- How to edit `config/sources.md` later (just push to main — the cloud trigger picks up changes automatically since it pulls fresh each run)
- How to pause/resume triggers
- Where to file issues upstream at `ryanschwartz88/the-edge` for skill bugs

## Guardrails

- Never commit secrets. Email address is fine (user-chosen). No API keys.
- Never create the repo private-by-default unless the user asked — default public matches the upstream.
- If any step fails, stop cleanly and tell the user exactly what to fix. Don't continue with a half-configured setup.
- If the user already has a repo at `{user}/{repo_name}`, confirm before any destructive action.

## Re-running

Running `edge-initialize` a second time on an already-initialized repo should be idempotent: re-interview only the fields the user wants to change, re-verify triggers exist, and send a fresh test email. Never recreate the repo.
