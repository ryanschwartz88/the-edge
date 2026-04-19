---
name: edge-refine
description: >
  Weekly source-quality pass for The Edge. Evaluates whether current sources
  in sources.md are producing signal, discovers new primary sources based on
  themes surfacing in recent digests, and either edits the config directly
  (local mode) or opens a pull request (repo mode). Never destructive — all
  changes use comment markers so they're reversible. Use when the user says
  "refine the edge", "tune edge sources", or when invoked by a weekly trigger.
allowed-tools: Bash Read Write Edit Glob Grep WebFetch WebSearch mcp__claude_ai_Gmail__*
---

# Edge Refine — Weekly Source Tuning

Evaluate `sources.md` against the last 4 weeks of real results. Propose drops and adds with rationale. Always reversible — drops are commented out, not hard-deleted.

## Philosophy

A source list must decay with intentional pruning. Good sources start publishing noise. Silent sources waste cycles. Themes drift. This skill keeps the config tuned without letting it stray from primary-source discipline.

## Edge home resolution

Same as `the-edge`:
1. `$THE_EDGE_HOME` if set.
2. CWD if it has `config/sources.md`.
3. `$HOME/.the-edge/`.

Call the resolved path `$EDGE_HOME`.

Detect **mode** by checking for `.git` at `$EDGE_HOME`:
- **local mode** — no `.git`. Edit `config/sources.md` in place. Email a diff summary.
- **repo mode** — `.git` present. Branch + PR flow.

## Inputs

- `$EDGE_HOME/config/sources.md` — current config.
- `$EDGE_HOME/data/digests/*.md` — last 4 weeks of digests.

## Pipeline

### 1. Gather recent digests

List `$EDGE_HOME/data/digests/*.md` dated within the last 28 days. For each digest, parse:
- Items that appeared (source domain, category, title).
- Rejection-reason counts in the footer.

Also read the prior 8 weeks if present for a longer rolling view.

### 2. Per-source tally

For each source URL in `config/sources.md`:
- `kept_count_4w` — items from this source that made the digest in the last 4 weeks.
- `kept_count_12w` — same over 12 weeks.
- Qualitative read: were the items substantive? (judge from titles + "why" lines in past digests).

### 3. Evaluate current sources

For each source, classify:

- **Keep** — `kept_count_4w ≥ 1` OR `kept_count_12w ≥ 3`.
- **Drop candidate** — `kept_count_4w == 0` AND `kept_count_12w == 0`, and not `# pinned`.
- **Watch** — silent this month but productive before.

### 4. Discover new sources

Scan the last 4 weeks of digests for recurring themes in titles and "why" lines that aren't already well-covered.

For each theme (top 3–5):
- Run 2 web searches for primary sources:
  - `"{theme}" research blog OR changelog OR release notes`
  - `"{theme}" site:arxiv.org` or similar primary-domain restriction
- For each candidate URL, fetch. Confirm primary (researcher, lab, company engineering blog, official docs) — not press, not aggregator.
- Check cadence (need 1+ post in last 30 days).
- Keep top candidates with one-line rationale each.

### 5. Apply the changes

#### Local mode

1. Back up the current config to `$EDGE_HOME/data/.sources-backup-YYYY-MM-DD.md` (so the user can restore easily).
2. Edit `$EDGE_HOME/config/sources.md` in place:
   - Drops: comment out the URL with `# dropped YYYY-MM-DD — {reason}` prefix.
   - Adds: append under the right category with `# added YYYY-MM-DD — {rationale}` on the line above.
3. Compute a unified diff of the two versions for the email.

#### Repo mode

1. `cd $EDGE_HOME`
2. Create branch `edge-refine/YYYY-MM-DD`.
3. Edit `config/sources.md` with the same drop/add markers.
4. Commit: `refine: {N} drops, {M} adds · YYYY-MM-DD`.
5. Push the branch.
6. `gh pr create --base main --head edge-refine/YYYY-MM-DD --title "Edge Refine · YYYY-MM-DD · {N} drops, {M} adds" --body <rationale>`.

### 6. Summary email

Send via Gmail MCP to `delivery.email` from config.

**Local mode email:**

```
Subject: Edge Refine · YYYY-MM-DD · {N} drops, {M} adds applied

Applied to $EDGE_HOME/config/sources.md. Backup at data/.sources-backup-YYYY-MM-DD.md.

Drops ({N}):
- {source} — no signal in 12w
- ...

Adds ({M}):
- {source} — {rationale}
- ...

Watch ({K}):
- {source} — silent 4w, productive before
- ...

Diff:
{unified-diff}

To revert: cp data/.sources-backup-YYYY-MM-DD.md config/sources.md
```

**Repo mode email:**

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
- ...

Review and merge the PR to apply, or close to ignore.
```

## Guardrails

- Never hard-delete a source — always comment with `# dropped`.
- Never propose more than **5 adds** or **3 drops** per run.
- Never touch a `# pinned` source.
- If fewer than 7 digests exist in the last 4 weeks (too little data), skip this run and send a one-line email explaining why.
- Repo mode: never merge the PR. User reviews.
- Local mode: always write a backup before editing.
