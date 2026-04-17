---
name: the-edge
description: >
  Build and send the daily primary-source news digest. Reads config/sources.md,
  fetches from primary sources (arXiv, research blogs, release notes, SEC
  filings), filters hype, picks the 10-15 highest-signal items, emails the
  result, and commits it to data/digests/. Use when the user says "run the
  edge", "morning brief", "send today's digest", or when invoked by the daily
  remote trigger.
allowed-tools: Bash Read Write Edit Glob Grep WebFetch WebSearch mcp__claude_ai_Gmail__*
---

# The Edge — Daily Primary-Source Digest

Build a daily digest of news from **primary sources only**. Reject curator and commentator content. Reward rarity and direct relevance to the configured topics.

## Philosophy

If everyone is reading it, there is no edge. This skill ignores aggregators (Techmeme, Stratechery, TechCrunch) as content and reads raw output instead: papers, changelogs, release notes, researcher blogs, regulator filings. Aggregators like Hacker News are used for **discovery** (extract the primary links they share), not for content.

## Inputs

- `config/sources.md` — source of truth for topics, sources, filter rules, delivery settings.
- Today's date in the timezone from config.
- The last 3 committed digests in `data/digests/` (for dedup against recent days).

## Flags

- `--dry-run` — print digest to stdout, do not send email, do not commit.
- `--no-email` — skip email, still commit.
- `--no-commit` — skip commit, still email.

Default (no flags): full run — fetch, filter, pick, email, commit, push.

## Pipeline

Do these steps in order. Announce each step briefly so the user can follow.

### 1. Load config

Read `config/sources.md`. Parse the frontmatter and the source lists under each category heading. If the file is missing or malformed, stop and tell the user to run `edge-initialize`.

### 2. Fetch candidates in parallel

Kick off these fetches concurrently — use multiple tool calls in a single message:

- **RSS/Atom feeds** — for each feed URL under "Primary sources" in config: `WebFetch` the feed URL. 10-second timeout per feed. Log failures, keep going.
- **Hacker News** — fetch `https://hacker-news.firebaseio.com/v0/topstories.json`, pull top 50 item details. Keep only stories with an external `url` (drop Ask/Show HN). Treat the linked URL as the candidate, not the HN page.
- **GitHub Trending** — `WebFetch https://github.com/trending/<lang>?since=daily` for the languages relevant to the user's stack (inferred from config topics).
- **Web search per topic** — for each topic in config, run 1–2 web searches restricted to primary-source domains (e.g., `site:arxiv.org`, `site:anthropic.com/research`).

For each candidate, capture: title, url, source name, source category, published date, short excerpt.

Cap total candidates at 200 before the next step — drop oldest first if over.

### 3. Dedupe

- Collapse items with identical canonical URLs.
- Collapse near-duplicate titles (same story from two sources).
- Drop any item whose URL already shipped in the last 3 digests in `data/digests/`.

### 4. Single-pass judgment

In one LLM pass, look at all remaining candidates and produce the final digest. Apply the hype filter rules from config — reject anything matching. Then pick the 10–15 highest-signal items using these heuristics in priority order:

1. **Primary source** (arxiv, official research blog, release notes, SEC, regulator filing, researcher blog) beats aggregator/press on the same story.
2. **Direct relevance** to a configured topic beats broad industry news.
3. **Rarity** — if you've seen this story on 5 sites, prefer the primary version and skip the rest.
4. **Applicability** — can the user plausibly use this in their work this week? Shipping code, security posture, architecture, or strategic decisions.

Track rejection counts by reason (`funding-only`, `marketing fluff`, `keynote recap`, etc.) for the footer.

No numeric scoring. No per-category caps beyond `limits.max_per_category` from config. Trust the judgment.

### 5. Render digest

Group by category in this order (skip empty categories). Use the emoji + label pattern the user configured; defaults:

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

If not `--no-email`: send via Gmail MCP (`mcp__claude_ai_Gmail__*`). To: `delivery.email` from config. Subject: `The Edge · YYYY-MM-DD · {N} items`. Body: the rendered markdown converted to HTML (headings, links clickable), with a plain-text fallback.

### 7. Commit + push

If not `--no-commit`: write markdown to `data/digests/YYYY-MM-DD.md`, `git add`, commit with message `digest: YYYY-MM-DD ({N} items)`, and `git push origin main`.

When invoked from a remote trigger, push uses whatever credentials are configured on the triggering environment. See the repo README for trigger setup.

## Error handling

- RSS fetch failure: log, continue. Include in "Fetch health" footer.
- Gmail send failure: commit still happens, surface the error.
- Git push failure (e.g., conflict): commit locally; stop and surface. Never force-push.
- Config missing or malformed: stop immediately, do not send a half-built digest.

## Interactive runs

When invoked interactively (not by a trigger), print short status lines between phases:

```
fetched: 43 rss, 18 hn, 7 gh-trending, 22 web = 90 candidates
dedupe: 76 unique
judgment: 12 kept, 41 rejected
```

Then show the digest and ask before sending/committing unless flags override.

## Stateless execution

This skill writes nothing outside the current working repo. Remote triggers invoke it with a fresh clone, let it commit + push, then discard the clone. That means `data/digests/` in the repo is the only persistent state. Don't rely on temp files or env-specific paths.
