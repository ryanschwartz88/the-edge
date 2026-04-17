---
name: edge-refine
description: >
  Weekly source-quality pass for The Edge. Evaluates whether current sources in
  config/sources.md are producing signal, discovers new primary sources based on
  themes surfacing in recent digests, and opens a pull request with a proposed
  config diff. Never merges — user reviews. Use when the user says "refine the
  edge", "tune edge sources", or when invoked by the weekly remote trigger.
allowed-tools: Bash Read Write Edit Glob Grep WebFetch WebSearch mcp__claude_ai_Gmail__*
---

# Edge Refine — Weekly Source Tuning

Evaluate `config/sources.md` against the last 4 weeks of real results. Propose drops and adds with rationale. Never edits the live config directly — always opens a PR.

## Philosophy

A source list must decay with intentional pruning. Good sources start publishing noise. Silent sources waste cycles. Themes drift. This skill keeps the config tuned without letting it stray from primary-source discipline.

## Inputs

- `config/sources.md` — current config.
- `data/digests/*.md` — the last 4 weeks of digests.

## Pipeline

### 1. Gather recent digests

List `data/digests/*.md` dated within the last 28 days. For each digest, parse:
- Items that appeared (their source domain, category, title).
- Rejection-reason counts in the footer.

Also read the prior 8 weeks if present for a longer rolling view — but the primary evaluation window is 4 weeks.

### 2. Per-source tally

For each source URL in `config/sources.md`:
- `kept_count_4w` — items from this source that made the digest in the last 4 weeks.
- `kept_count_12w` — same over 12 weeks (rolling context).
- A qualitative read of whether the items that surfaced were themselves substantive (the refiner can judge from titles + the "why" lines in past digests).

### 3. Evaluate current sources

For each source, classify:

- **Keep** — `kept_count_4w ≥ 1` OR `kept_count_12w ≥ 3`.
- **Drop candidate** — `kept_count_4w == 0` AND `kept_count_12w == 0`, and the source is not `# pinned`.
- **Watch** — silent this month but productive before; note, don't drop.

### 4. Discover new sources

Scan the last 4 weeks of digests for recurring themes the user clearly cares about (topics mentioned in titles or "why" lines that aren't already well-covered by existing sources).

For each theme (top 3–5):
- Run 2 web searches aimed at primary sources:
  - `"{theme}" research blog OR changelog OR release notes`
  - `"{theme}" site:arxiv.org` or similar primary-domain restriction
- For each candidate URL, fetch the page. Confirm it's primary (researcher, lab, company engineering blog, official docs) — not press, not aggregator.
- Check publishing cadence (need 1+ post in the last 30 days).
- Keep the top candidates with a one-line rationale each.

### 5. Propose the diff

On a new branch `edge-refine/YYYY-MM-DD`:

- Edit `config/sources.md`:
  - Comment out drops with `# dropped YYYY-MM-DD — {reason}` (do not hard-delete, so it's easy to revert).
  - Add new sources under the right category with `# added YYYY-MM-DD — {rationale}`.
- Commit with message `refine: {N} drops, {M} adds · YYYY-MM-DD`.
- Push the branch.
- Open a PR against `main` using `gh pr create` with:
  - Title: `Edge Refine · YYYY-MM-DD · {N} drops, {M} adds`
  - Body: summary of drops, adds, and any watch-list notes. Include rationale per entry.

### 6. Summary email

Send via Gmail MCP to `delivery.email` from config:

```
Subject: Edge Refine · YYYY-MM-DD · {N} drops, {M} adds proposed

PR: {pr_url}

Drops ({N}):
- {source} — no signal in 12w
- ...

Adds ({M}):
- {source} — {rationale}
- ...

Watch ({K}):
- {source} — silent 4w, productive before
- ...

Review and merge the PR to apply, or close to ignore.
```

## Guardrails

- Never hard-delete from config — comment with `# dropped`.
- Never propose more than **5 adds** or **3 drops** in a single run.
- Never propose dropping a `# pinned` source.
- If fewer than 7 digests exist in the last 4 weeks (too little data), skip this run and send a one-line email explaining why.
- Do not merge the PR. The user reviews.

## Stateless execution

This skill is safe to run from a fresh clone via remote trigger. All state is in the repo: `config/sources.md` and `data/digests/`. The PR itself is the delivery mechanism.
