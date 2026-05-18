# SEO + content strategy

The goal of `hugonissar.github.io` is to drive traffic to
**[github.com/hugonissar](https://github.com/hugonissar)** and the two
open-source repositories there. Every choice in the site's technical SEO
setup, content templates, and editorial calendar points back to that.

This document is the playbook. Read it once, refer back to it before
publishing each post.

---

## The mental model

A typical visitor's journey:

```
Google / AI search   →   Blog post on this site   →   Click through to repo   →   Star + clone
   (a query)             (with embedded CTA)            (on github.com)            (the conversion)
```

Two things make that journey work:

1. **The query has to find the post.** That's traditional SEO and AI-search
   optimization — schema, headings, freshness, citations, structure.
2. **The post has to push to GitHub.** That's content design — every post
   names the repo, links to it multiple times with different anchor text,
   and ends with a CTA.

Everything below serves one of those two goals.

---

## Keyword universe

Pulled directly from the keywords already in the repo READMEs, plus
adjacent queries real people search for.

### Primary keywords (high intent, high relevance)

| Cluster | Example queries | Search intent |
|---|---|---|
| **BigQuery MCP** | `bigquery mcp server`, `claude bigquery integration`, `mcp server cloud run`, `bigquery mcp self-hosted`, `read-only bigquery mcp` | Engineers building agents that need BigQuery access |
| **GA4 propensity** | `ga4 purchase propensity model`, `bqml purchase propensity`, `ga4 predictive audience alternative`, `google ads remarketing bigquery ml` | Marketing analysts / engineers building remarketing |
| **GA4 + BQ plumbing** | `ga4 measurement protocol python`, `ga4 user property bigquery`, `ga4 bigquery export tutorial`, `ga4 consent mode v2 bigquery` | MarTech engineers wiring GA4 → BQ → Ads |
| **BQML tutorials** | `bigquery ml boosted tree classifier`, `bqml hyperparameter tuning`, `ml.predict tutorial`, `bigquery ml feature importance` | Analytics engineers learning BQML |
| **Cloud Run / Functions for marketing** | `cloud run mcp server`, `cloud function ga4 measurement protocol`, `cloud scheduler bigquery ml` | Devs deploying scheduled Cloud jobs |

### Secondary keywords (lower volume, easier wins)

- `bigquery cost control ai`
- `bigquery rate limit ai agent`
- `bigquery scan budget`
- `model armor alternative`
- `mcp server allowlist`
- `consent mode v2 bigquery ml`
- `ga4 audience automation`
- `ga4 user property write back`

### Branded / discovery queries

- `hugo nissar`
- `hugonissar github`
- `BigQuery-Read-Only-MCP-Server`
- `GA4-Ecommerce-BQML-Purchase-Propensity`

The Person schema in `_includes/json-ld-person.html` plus the
`<link rel="me">` in the footer establish you as an entity in Google's
Knowledge Graph over time. This is what makes name-based queries find you.

---

## Content formats that actually get cited

Per the Princeton GEO research and citation-share data:

| Format | Share of AI citations | Use it for |
|---|---|---|
| **Comparison posts** | ~33% | "X vs Y", "managed vs self-hosted", "framework A vs B" |
| **Definitive guides** | ~15% | "How to do X end-to-end" — the canonical reference |
| **Original research / data** | ~12% | Benchmarks, cost analyses, latency tests |
| **Listicles / best-of** | ~10% | "10 things to know about Y" |
| **How-to guides** | ~8% | Step-by-step tutorials |

The two seed posts ship cover the first two formats. Keep that ratio. Aim for
roughly **50% comparison content**, **30% definitive guides**, and **20%
everything else**.

---

## Editorial calendar — the next 10 posts

Each of these has a clear search intent and a clear repo to drive traffic to.
Use the seed posts as templates.

| # | Title | Format | Target query | Repo CTA |
|---|---|---|---|---|
| 1 | Self-hosted vs Google's official BigQuery MCP server | Comparison | `bigquery mcp server comparison` | BigQuery-Read-Only-MCP-Server |
| 2 | How to build a GA4 purchase propensity model with BigQuery ML | Guide | `ga4 purchase propensity bqml` | GA4-...-Propensity |
| 3 | Why BQML beats Vertex AI for single-model marketing pipelines | Comparison | `bqml vs vertex ai` | GA4-...-Propensity |
| 4 | How to push BQML predictions back to GA4 with the Measurement Protocol | Guide | `ga4 measurement protocol user property bqml` | GA4-...-Propensity |
| 5 | Stopping the $2,000 AI query: cost ceilings for BigQuery MCP | Guide | `bigquery cost control mcp` | BigQuery-Read-Only-MCP-Server |
| 6 | Consent Mode v2 + BigQuery ML: where to apply the consent filter | Comparison | `consent mode v2 bigquery ml` | GA4-...-Propensity |
| 7 | Allowlists vs IAM: how to lock down a BigQuery MCP server | Guide | `bigquery mcp security` | BigQuery-Read-Only-MCP-Server |
| 8 | Inside a BQML boosted-tree purchase propensity model: feature importances | Original research | `bqml feature importance propensity` | GA4-...-Propensity |
| 9 | GA4 BigQuery export schema for ML: the fields that actually matter | Guide | `ga4 bigquery export schema ml` | GA4-...-Propensity |
| 10 | Connecting Claude Desktop to BigQuery with MCP in 5 minutes | Tutorial | `claude desktop bigquery mcp tutorial` | BigQuery-Read-Only-MCP-Server |

**Cadence:** one post every 1–2 weeks. Bursts of three in a week are fine.
Long gaps are not — freshness is a citation signal.

---

## Post structure — the template that works

Every post should follow this template. The two seed posts demonstrate it:

1. **Lead with a one-sentence summary** (a self-contained answer the AI can
   extract). Don't bury the lede.
2. **TL;DR table or bullet list.** AI extractors love this.
3. **Definition / "What this is" paragraph** if it's a topic post.
4. **Comparison table** if it's a comparison post (with `| | A | B |` header).
5. **Body sections with H2 headings phrased as questions** people actually
   search for. *Why this matters, when to use it, how to deploy it.*
6. **Code blocks** with real, runnable commands.
7. **FAQ section** with 4–8 H3 questions at the end. This is the single most
   citable section in any post.
8. **CTA back to the repo.** Multiple anchor variants: *"View on GitHub"*,
   *"the full source is at github.com/..."*, *"PRs welcome"*.

Word count target: **1,500–2,500 words.** Short enough to read in one sitting,
long enough to be the definitive answer.

---

## Technical SEO foundations (already shipped)

These are configured in the templates. You don't need to do anything; just
don't break them.

| Foundation | Implementation | Where |
|---|---|---|
| Sitemap | Auto-generated, listed in robots.txt + linked in `<head>` | jekyll-sitemap |
| RSS feed | All posts, 50-post limit | jekyll-feed |
| `robots.txt` | Allows all AI bots explicitly (GPTBot, ClaudeBot, PerplexityBot, Google-Extended, ...) | `/robots.txt` |
| `llms.txt` | AI-agent context per llmstxt.org | `/llms.txt` |
| Canonical URLs | Auto-generated; explicit `url:` in _config.yml | jekyll-seo-tag |
| Open Graph + Twitter cards | Auto-generated for every page | jekyll-seo-tag |
| `BlogPosting` schema | On every blog post | jekyll-seo-tag |
| `Person` schema | On every page | `_includes/json-ld-person.html` |
| `SoftwareSourceCode` schema | On every project page | `_layouts/project.html` |
| `last_modified_at` | Per post via front matter — freshness signal | `_layouts/post.html` |
| Mobile-friendly | Responsive CSS, viewport meta | `_layouts/default.html` + `main.scss` |
| Fast LCP | No JS, minimal CSS, system font stack | `main.scss` |
| Dark mode | `prefers-color-scheme` | `main.scss` |
| HTTPS | Enforced by GitHub Pages | github.com setting |
| HTMLProofer in CI | Catches broken internal links | `deploy.yml` |

---

## Things to do once the site is live

In order of impact:

1. **Submit `sitemap.xml` to Google Search Console.** Two minutes, big payoff.
2. **Submit to Bing Webmaster Tools.** Same URL, free, takes another two
   minutes. Powers Copilot citations.
3. **Add the site URL to your GitHub profile.** Settings → public profile →
   website. Adds an inbound link from a high-authority domain.
4. **Cross-link from each repo's README** back to the relevant project page
   on the site (e.g. add *"Full writeup: hugonissar.github.io/projects/..."*
   to each repo's README's "About" section).
5. **Submit posts to relevant aggregators** when you have 3–4 published:
   Hacker News, Lobsters, r/bigquery, r/MarketingAnalytics, r/ProgrammerHumor
   only if they really fit. *Don't* spam — one well-timed post on HN beats
   ten low-effort posts on niche subreddits.
6. **Run AI visibility checks monthly.** Pick 10 queries from the keyword
   table above. Run each through ChatGPT, Perplexity, Google AI Overviews,
   Claude (with search). Log who's cited. Iterate.

---

## Monitoring

Free tools that cover 90% of what you need:

| Tool | What it tells you | URL |
|---|---|---|
| Google Search Console | Impressions, clicks, query data | https://search.google.com/search-console |
| Bing Webmaster Tools | Bing/Copilot equivalent | https://www.bing.com/webmasters |
| GitHub Insights (per repo) | Referral sources, clones | github.com/hugonissar/REPO/graphs/traffic |
| PageSpeed Insights | Core Web Vitals | https://pagespeed.web.dev/ |

GitHub repo traffic is the **direct measure** of whether the strategy is
working. The site exists to move that number.

---

## Anti-patterns — don't do these

- **Don't gate any content.** AI crawlers can't read past gates.
- **Don't keyword-stuff.** Princeton GEO showed it *reduces* citation
  probability by ~10%.
- **Don't write thin posts.** AI extractors weight comprehensive coverage.
- **Don't ignore freshness.** Refresh popular posts every 6 months and bump
  `last_modified_at`. AI systems weight recency.
- **Don't forget the repo CTA.** Every post must drive somewhere on GitHub.
- **Don't lie in titles.** If a post says "5-minute guide," it had better
  be a 5-minute guide. Click-through and dwell-time signals matter.
- **Don't post-and-forget.** Distribution beats production. After publishing,
  share on HN/Reddit/X/LinkedIn at least once. One tweet at the right time
  beats five posts no one sees.
