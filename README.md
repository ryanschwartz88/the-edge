# The Edge

A daily primary-source news digest delivered to your inbox. Local-first, cloud-optional.

Four skills:

- **`edge-initialize`** — local first-time setup. Interview, write `~/.the-edge/`, send a test digest. No GitHub required.
- **`the-edge`** — the daily run. Fetches primary sources, filters hype, picks 10–15 items worth your time, emails you, writes the digest to `~/.the-edge/data/digests/`.
- **`edge-refine`** — weekly source tuning. Drops dead sources, proposes new primary sources based on themes surfacing in your digests. Reversible (comment markers, local backups).
- **`edge-remote`** — *opt-in.* Promote local setup to a cloud-scheduled trigger so the digest arrives with your laptop closed.

## Philosophy

If everyone is reading it, there is no edge. This skill ignores aggregators (Techmeme, Stratechery, TechCrunch) as content and reads raw output instead — papers, changelogs, release notes, researcher blogs, regulator filings. Aggregators like Hacker News are used for _discovery_ (extract primary links they share), not as content.

The source list decays over time. `edge-refine` prunes dead sources and proposes new ones based on themes surfacing in your actual reading.

## Install

One command installs all four skills into your agent:

```bash
npx skills add ryanschwartz88/the-edge -a claude-code --global
```

Works with Claude Code, Cursor, OpenCode. Swap `-a` for your agent.

## Quickstart — local only (no GitHub needed)

Start your agent anywhere and run:

```
edge-initialize
```

It will:
1. Interview you for email, timezone, topics, stack keywords.
2. Write `~/.the-edge/config/sources.md` personalized to your lens.
3. Print a test digest (dry run) so you see what you'd get.
4. Send a real test email to confirm Gmail MCP works.
5. Optionally register a daily `launchd` trigger (macOS) so it runs every morning while your laptop is awake.

Prerequisites:
- Gmail MCP configured in your agent.
- (macOS launchd option only) a Mac.

That's it. No fork, no repo, no `gh`.

## Daily use

Nothing. If you registered the `launchd` trigger, the digest arrives on schedule. A markdown copy lands in `~/.the-edge/data/digests/YYYY-MM-DD.md`.

To preview without sending:

```
the-edge --dry-run
```

Other flags:
- `--no-email` — write the digest file but skip email.
- `--no-commit` — email + write the file but skip any git activity (only matters in repo mode).
- `--here` — force use of the current directory as the edge home.

## Weekly tuning

`edge-refine` reviews the last 4 weeks of digests, drops dead sources, and proposes new primary sources for themes that are surfacing. In local mode it edits `~/.the-edge/config/sources.md` directly (always with a backup) and emails a diff summary. Run it manually whenever:

```
edge-refine
```

All changes use `# added YYYY-MM-DD` / `# dropped YYYY-MM-DD` comment markers and a timestamped backup in `~/.the-edge/data/` — revert with one copy.

## Going cloud-scheduled (optional)

If you want the digest to arrive with your laptop closed, run:

```
edge-remote
```

This creates a **private** GitHub repo from your `~/.the-edge/` contents and wires up a daily trigger (for `the-edge`) and a weekly trigger (for `edge-refine`) via the `schedule` / `RemoteTrigger` system. After it runs, close your laptop — the emails keep coming.

Prerequisites: `gh` CLI authenticated (`gh auth login`), the `schedule` feature available in your agent.

In repo mode `edge-refine` opens a PR instead of editing in place — same rationale, different delivery.

## Customizing your sources

Edit `~/.the-edge/config/sources.md` any time. If you're in local mode, changes take effect on the next run. If you've gone cloud (`edge-remote`), edit and `git push` (or use GitHub's web editor); the next cloud run pulls fresh.

Guidelines:
- **Primary sources only.** Researcher blogs, lab research pages, changelogs, release feeds, SEC filings. Not press, not aggregators.
- **Specific topics.** "AI research" is too vague. "LLM agents, evals, RAG, prompt injection" is specific.
- **`# pinned`** marks sources that `edge-refine` must never drop.

## How state resolution works

Every skill resolves the state directory in this order:

1. `$THE_EDGE_HOME` if set (override).
2. Current working directory if it contains `config/sources.md` (repo mode — used by cloud triggers running inside a cloned state repo).
3. `$HOME/.the-edge/` (local default).

**Mode** is detected from the presence of `.git`:
- No `.git` → local mode. Files only, no git activity.
- `.git` present → repo mode. Daily runs commit+push; refine uses PR flow.

## Repo layout

```
the-edge/                                  (distribution repo, this one)
├── config/
│   └── sources.md                         ← starter template copied by edge-initialize
├── data/
│   └── digests/.gitkeep
├── skills/
│   ├── the-edge/SKILL.md
│   ├── edge-refine/SKILL.md
│   ├── edge-initialize/SKILL.md
│   └── edge-remote/SKILL.md
├── LICENSE
├── package.json
└── README.md
```

After `edge-initialize`, your personal state lives in:

```
~/.the-edge/
├── config/
│   └── sources.md
└── data/
    └── digests/
        └── YYYY-MM-DD.md
```

## License

MIT. Fork freely.
