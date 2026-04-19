---
name: the-edge
description: >
  Build and send the daily primary-source news digest. Reads sources.md from
  the user's edge home (default ~/.the-edge/), fetches from primary sources
  (arXiv, research blogs, release notes, SEC filings), filters hype, picks the
  10-15 highest-signal items, emails the result, writes a dated digest file,
  and — if the edge home is a git checkout — commits and pushes. Use when the
  user says "run the edge", "morning brief", "send today's digest", or when
  invoked by a remote trigger.
allowed-tools: Bash Read Write Edit Glob Grep WebFetch WebSearch mcp__claude_ai_Gmail__*
---

# The Edge — Daily Primary-Source Digest

Build a daily digest of news from **primary sources only**. Reject curator and commentator content. Reward rarity and direct relevance to the configured topics.

## Philosophy

If everyone is reading it, there is no edge. This skill ignores aggregators (Techmeme, Stratechery, TechCrunch) as content and reads raw output instead: papers, changelogs, release notes, researcher blogs, regulator filings. Aggregators like Hacker News are used for **discovery** (extract the primary links they share), not for content.

## Edge home resolution

Before anything else, resolve the edge home directory in this order:

1. If `$THE_EDGE_HOME` is set, use it.
2. If the current working directory contains `config/sources.md`, use CWD (repo mode, typical when invoked by a cloud trigger against a cloned state repo).
3. Otherwise, use `$HOME/.the-edge/`.

If none of these resolve to a directory with `config/sources.md`, stop and tell the user to run `edge-initialize` first.

Detect **mode** by checking for `.git` at the edge home root:
- **local mode** — no `.git`. Files only; no git activity at end of run.
- **repo mode** — `.git` present. Commit and push the digest at end of run.

From here, `$EDGE_HOME` refers to the resolved edge home directory.

## Inputs

- `$EDGE_HOME/config/sources.md` — source of truth for topics, sources, filter rules, delivery settings.
- Today's date in the timezone from config.
- The last 3 digests in `$EDGE_HOME/data/digests/` (for dedup against recent days).

## Flags

- `--dry-run` — print digest to stdout; no email, no file write, no git.
- `--no-email` — skip email; still write the digest file and (if repo mode) commit+push.
- `--no-commit` — skip commit+push; still email and write the digest file.
- `--here` — force use of CWD as the edge home (override resolution).

Default (no flags): full run — fetch, filter, pick, email, write, and (if repo mode) commit+push.

## Pipeline

Do these steps in order. Announce each step briefly so the user can follow.

### 1. Load config

Read `$EDGE_HOME/config/sources.md`. Parse the frontmatter and source lists. If malformed, stop and surface the error.

### 2. Fetch candidates in parallel

Kick off these fetches concurrently — use multiple tool calls in a single message:

- **RSS/Atom feeds** — for each feed URL under "Primary sources" in config: `WebFetch` the feed URL. 10-second timeout per feed. Log failures, keep going.
- **Hacker News** — fetch `https://hacker-news.firebaseio.com/v0/topstories.json`, pull top 50 item details. Keep only stories with an external `url` (drop Ask/Show HN). Treat the linked URL as the candidate, not the HN page.
- **GitHub Trending** — `WebFetch https://github.com/trending/<lang>?since=daily` for languages relevant to the user's stack (inferred from config topics).
- **Web search per topic** — for each topic in config, run 1–2 web searches restricted to primary-source domains (e.g., `site:arxiv.org`, `site:anthropic.com/research`).

For each candidate capture: title, url, source name, source category, published date, short excerpt.

Cap total candidates at 200 before the next step — drop oldest first if over.

### 3. Dedupe

- Collapse items with identical canonical URLs.
- Collapse near-duplicate titles (same story from two sources).
- Drop any item whose URL already shipped in the last 3 digests in `$EDGE_HOME/data/digests/`.

### 4. Single-pass judgment

In one LLM pass, look at all remaining candidates and produce the final digest. Apply the hype filter rules from config — reject anything matching. Then pick the 10–15 highest-signal items using these heuristics in priority order:

1. **Primary source** (arxiv, official research blog, release notes, SEC, regulator filing, researcher blog) beats aggregator/press on the same story.
2. **Direct relevance** to a configured topic beats broad industry news.
3. **Rarity** — if you've seen the same story on 5 sites, prefer the primary version and skip the rest.
4. **Applicability** — can the user plausibly use this in their work this week? Shipping code, security posture, architecture, or strategic decisions.

Track rejection counts by reason (`funding-only`, `marketing fluff`, `keynote recap`, etc.) for the footer.

No numeric scoring. Respect `limits.max_items` and `limits.max_per_category` from config; otherwise trust the judgment.

### 5. Render digest

Group by category in this order (skip empty categories):

1. Models & Labs
2. Tooling & Orchestration
3. Security
4. Mobile / Platform
5. Markets & Strategy
6. Deep Dive (optional — 1 long-read worth real time)

Per item:

```
- **{title}** ({domain})
  {1-line reason this matters, grounded in the user's work}
  [link]({url})
```

Footer:
```
─── Filtered {N} items ───
{reason1}: {count1} · {reason2}: {count2} · ...

─── Fetch health ───
{M} sources unreachable: {list}
```

### 6. Email

If not `--no-email`: send via Gmail MCP (`mcp__claude_ai_Gmail__*`). To: `delivery.email` from config. Subject: `The Edge · YYYY-MM-DD · {N} items`. Body: rendered markdown converted to HTML (headings, links clickable), with a plain-text fallback.

### 7. Write digest file

If not `--dry-run`: write markdown to `$EDGE_HOME/data/digests/YYYY-MM-DD.md`. Overwrite if it already exists (re-runs of the same day replace, not append).

### 8. Commit + push (repo mode only)

If repo mode and not `--no-commit` and not `--dry-run`:

```bash
cd $EDGE_HOME
git add data/digests/YYYY-MM-DD.md
git commit -m "digest: YYYY-MM-DD ({N} items)"
git push origin main
```

If the commit fails (no changes) or push fails (conflict), surface the error. Never force-push.

In local mode this step is skipped entirely.

## Error handling

- RSS fetch failure: log, continue. Include in "Fetch health" footer.
- Gmail send failure: file write still happens, surface the error.
- Git push failure: commit is already local; stop and surface. Never force-push.
- Config missing or malformed: stop immediately, do not send a half-built digest.

## Interactive runs

When invoked interactively (not by a trigger), print short status lines between phases:

```
edge-home: /Users/you/.the-edge (local mode)
fetched:   43 rss, 18 hn, 7 gh-trending, 22 web = 90 candidates
dedupe:    76 unique
judgment:  12 kept, 41 rejected
```

Then show the digest and ask before sending / writing unless flags override.
