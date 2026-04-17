# The Edge

A daily primary-source news digest that runs in the cloud and emails you every morning.

Three skills:

- **`edge-initialize`** — one-time setup. Forks this repo under your account, personalizes it, wires up cloud triggers, sends a test digest.
- **`the-edge`** — the daily run. Fetches primary sources, filters hype, picks 10–15 items worth your time, emails you, commits the digest.
- **`edge-refine`** — weekly source tuning. Drops sources that stopped producing, proposes new ones based on themes surfacing in your digests, opens a PR for review.

## Philosophy

If everyone is reading it, there is no edge. This skill ignores aggregators (Techmeme, Stratechery, TechCrunch) as content and reads raw output instead — papers, changelogs, release notes, researcher blogs, regulator filings. Aggregators like Hacker News are used for _discovery_ (extract primary links they share), not as content.

The source list decays over time. `edge-refine` prunes dead sources and proposes new ones based on themes surfacing in your actual reading.

## Install

One command installs all three skills into your agent:

```bash
npx skills add ryanschwartz88/the-edge -a claude-code --global
```

Works with Claude Code, Cursor, OpenCode. Swap `-a` for your agent.

## Setup (first time only)

In any directory, start your agent and run:

```
edge-initialize
```

It will:
1. Fork this repo to your GitHub account.
2. Interview you for email, timezone, topics, and stack keywords.
3. Rewrite `config/sources.md` for your lens.
4. Create a daily remote trigger (cloud-scheduled — your laptop can be closed).
5. Create a weekly remote trigger for `edge-refine`.
6. Send a test digest as proof.

Prerequisites:
- `gh` CLI installed and authenticated (`gh auth login`).
- Gmail MCP configured in your agent.
- `git` configured with `user.name` and `user.email`.

## Daily use

Nothing. The cloud trigger runs the skill against your forked repo every morning. The digest arrives in your inbox. A markdown copy is committed to `data/digests/YYYY-MM-DD.md`.

To preview a digest interactively without sending/committing:

```
the-edge --dry-run
```

Other flags:
- `--no-email` — commit but skip email.
- `--no-commit` — email but skip commit.

## Weekly tuning

`edge-refine` runs every Sunday via its trigger. It opens a pull request on your repo proposing drops (dead sources) and adds (new primary sources for themes surfacing in your recent digests). You review, merge, or close. Never auto-merges.

To tune manually:

```
edge-refine
```

## Customizing your sources

Edit `config/sources.md` in your fork and push. The next daily trigger picks it up automatically (it pulls fresh each run).

Guidelines:
- **Primary sources only.** Researcher blogs, lab research pages, changelogs, release feeds, SEC filings. Not press, not aggregators.
- **Specific topics.** "AI research" is too vague. "LLM agents, evals, RAG, prompt injection" is specific.
- **Mark sources as `# pinned`** to prevent `edge-refine` from ever dropping them.

## Architecture

- **Stateless skills.** All persistent state is in your repo (`config/sources.md`, `data/digests/`). Remote triggers clone fresh each run, work, push back, discard the clone.
- **Nothing on your laptop.** Triggers run in Anthropic's cloud. Close your laptop, the digest still arrives.
- **Your fork is the state.** Past digests committed to git serve as dedup history and the refiner's input signal.

## Repo layout

```
the-edge/
├── config/
│   └── sources.md             # your topics, sources, filter rules
├── data/
│   └── digests/YYYY-MM-DD.md  # daily digests, committed as they ship
├── skills/
│   ├── the-edge/SKILL.md
│   ├── edge-refine/SKILL.md
│   └── edge-initialize/SKILL.md
├── LICENSE
└── README.md
```

## License

MIT. Fork freely.
