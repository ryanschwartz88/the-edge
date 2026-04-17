---
owner: YOUR_NAME
timezone: America/Los_Angeles
delivery:
  email: you@example.com
  send_time: "07:00"
limits:
  max_items: 12
  max_per_category: 4
---

# Sources

Drives `the-edge` (daily digest) and `edge-refine` (weekly source tuning). Edit everything in this file to match your lens. Every fork customizes this.

## Topics

3–6 themes you want news for. Be specific — vague topics produce vague digests.

- AI research — agents, evals, RAG, prompt injection, new model capabilities
- Tooling — Claude Code, MCP, Codex, local LLMs, orchestration
- Security — mobile reverse engineering, LLM red-teaming, relevant CVEs
- Mobile dev — iOS, Expo, React Native, Swift evolution
- Markets — primary filings and disclosures for companies you track

## Primary sources

Only list sources that **produce** content — researchers, labs, companies, regulators. Aggregators belong under Discovery.

### AI research
- https://arxiv.org/rss/cs.AI
- https://arxiv.org/rss/cs.LG
- https://huggingface.co/papers
- https://www.anthropic.com/research  # pinned
- https://openai.com/research  # pinned
- https://deepmind.google/research/
- https://metr.org/blog

### Researchers (max 1 item per digest per researcher)
- https://karpathy.github.io/
- https://simonwillison.net/atom/everything/
- https://www.latent.space/feed

### Tooling
- https://github.com/anthropics/claude-code/releases.atom  # pinned
- https://docs.claude.com/en/release-notes/claude-code
- https://platform.openai.com/docs/changelog
- https://www.anthropic.com/engineering

### Mobile dev
- https://expo.dev/changelog
- https://github.com/facebook/react-native/releases.atom
- https://github.com/apple/swift-evolution/commits/main.atom
- https://developer.apple.com/news/releases/rss/releases.rss

### Security
- https://googleprojectzero.blogspot.com/feeds/posts/default
- https://embracethered.com/blog/index.xml
- https://nvd.nist.gov/feeds/xml/cve/misc/nvd-rss.xml

### Markets
- https://www.ft.com/alphaville?format=rss
- # add SEC feeds for companies you track — EDGAR atom URLs: https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=XXXX&type=&dateb=&owner=include&count=40&output=atom

## Discovery (surface primary links, not content)

- Hacker News top stories — https://hacker-news.firebaseio.com/v0/topstories.json
- GitHub Trending — pick languages relevant to your stack
- Product Hunt daily

## Hype filter — hard reject rules

Reject any item that matches:

- Funding announcement without product/technical substance ("X raises $Y")
- "AI-powered [anything]" marketing framing
- Thought-leadership op-ed with no new data or shipped work
- Conference keynote recap with no new release
- "AI will [change / transform / disrupt] [industry]" future-tense speculation
- Personality news — who joined, who left, who got promoted
- Syndicated press releases (same wording on 3+ sites)
- Rumor/leak articles without primary source
- Listicle roundups ("Top N tools")

## Boost signal

Sources, keywords, or topics to actively favor. Nudge these up when scoring. Edit for your stack.

- # Replace with topics specific to your work — e.g., "AlarmKit", "SwiftUI", "MCP", "Convex"

## Markers (reserved — the refiner writes these)

- `# pinned` — the refiner will never propose dropping this source.
- `# added YYYY-MM-DD — {reason}` — the refiner added this source; review on next digest.
- `# dropped YYYY-MM-DD — {reason}` — the refiner dropped this source. Restore by removing the marker and uncommenting.
