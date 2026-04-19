---
name: edge-initialize
description: >
  First-time local setup for The Edge. Creates ~/.the-edge/ with a
  personalized sources.md and an empty digests directory, verifies Gmail MCP,
  runs a dry-run digest for sanity, and optionally registers a daily launchd
  trigger on macOS. No GitHub involved. Use when the user says "set up the
  edge", "initialize the edge", or has just installed the skills.
allowed-tools: Bash Read Write Edit Glob mcp__claude_ai_Gmail__*
---

# Edge Initialize — Local First-Time Setup

Turn a fresh install of these skills into a working, personalized daily brief. No GitHub, no fork. State lives in `~/.the-edge/`. After this, `the-edge` runs against that directory.

If the user later wants a laptop-closed cloud trigger, they run `edge-remote` — that's a separate skill.

## What this skill produces

1. `~/.the-edge/config/sources.md` — personalized with the user's email, topics, stack keywords, timezone.
2. `~/.the-edge/data/digests/` — empty directory ready to receive daily digests.
3. A test digest printed to stdout (via `--dry-run`) so the user sees it works.
4. Optionally: a `launchd` plist at `~/Library/LaunchAgents/com.the-edge.daily.plist` that fires the daily run at the user's chosen send time (macOS only).

## Prerequisites

Before starting, verify:

- Gmail MCP available — check that `mcp__claude_ai_Gmail__*` tools are present. If not, instruct the user to authenticate via `mcp__claude_ai_Gmail__authenticate`, then resume.
- `~/.the-edge/` does not already exist — if it does, ask the user whether to re-initialize (rewrites config, preserves digests) or cancel.

If any prereq fails, stop and surface exactly what's needed. Don't guess.

## Interview

Ask these questions one at a time (don't batch). Record answers:

1. **Email address** — where digests get delivered.
2. **Timezone** — e.g., `America/Los_Angeles`.
3. **Send time** — preferred daily delivery time (default `07:00`).
4. **Topics** — 3 to 6 themes, specific. Show the user the starter topics from the template and ask whether to keep, replace, or edit.
5. **Stack keywords** — boost-signal keywords tied to what they're building (e.g., `SwiftUI`, `MCP`, `Next.js`).

## Pipeline

### 1. Create the directory structure

```bash
mkdir -p ~/.the-edge/config ~/.the-edge/data/digests
```

### 2. Write the personalized config

Copy the starter template from this skill package's `config/sources.md` (bundled alongside the SKILL.md) to `~/.the-edge/config/sources.md`, then edit:

- Replace `owner: YOUR_NAME` with the user's OS username (or blank if unknown — it's informational).
- Replace `email: you@example.com` with the interview answer.
- Update `timezone` and `delivery.send_time`.
- Replace the `## Topics` list with the user's topics.
- Replace the `## Boost signal` placeholders with the user's stack keywords.
- Leave the primary-source lists as starter defaults. `edge-refine` will tune them over time.

### 3. Verify with a dry run

Invoke `the-edge --dry-run`. Confirm it prints a digest to stdout with ≥ 5 items, primary sources dominating, hype filter catching at least some items.

If zero items surface, the topic list is too narrow. Loop back to the interview and widen topics.

### 4. Test email

Invoke `the-edge --no-commit` (writes a digest file but still emails). Ask the user to confirm the email arrived. If not: likely a Gmail MCP auth issue. Surface and stop.

### 5. Optional: launchd trigger (macOS)

Ask: "Register a daily `launchd` trigger so your laptop runs the-edge every morning? (y/n)"

If yes, and the platform is darwin:

Write `~/Library/LaunchAgents/com.the-edge.daily.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.the-edge.daily</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-lc</string>
    <string>cd ~ &amp;&amp; claude --headless "run the the-edge skill"</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>{HOUR}</integer>
    <key>Minute</key>
    <integer>{MINUTE}</integer>
  </dict>
  <key>StandardOutPath</key>
  <string>/tmp/the-edge.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/the-edge.err</string>
</dict>
</plist>
```

Load it: `launchctl load ~/Library/LaunchAgents/com.the-edge.daily.plist`.

Tell the user: "Daily run scheduled for {HH:MM} {timezone}. The daily job needs your laptop to be awake — if you'd rather it fire from the cloud with your laptop closed, run `edge-remote` next."

On non-macOS, skip and suggest `edge-remote` instead.

### 6. Summary to user

Report:
- `~/.the-edge/` layout
- Where to edit `config/sources.md` later (just edit in place — no git needed in local mode)
- Whether a `launchd` trigger was registered, and at what time
- Next step: run `edge-remote` if they want cloud-scheduled runs laptop-closed

## Guardrails

- Never commit secrets. Email address and keywords are fine; no API keys.
- Never overwrite `~/.the-edge/` silently — ask first if it exists.
- If any step fails, stop cleanly and tell the user exactly what to fix. Don't continue with half-configured setup.

## Re-running

Re-running `edge-initialize` on an existing setup should only rewrite `config/sources.md` after confirming with the user. Never delete `data/digests/`. Never overwrite the `launchd` plist without asking.
