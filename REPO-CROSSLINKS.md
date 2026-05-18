# Cross-linking your existing repos back to the site

This is the second half of the SEO loop. The site links **to** your repos.
For the maximum traffic effect, the repos should also link **back to the
site** — specifically to the deep-dive blog post about each one.

Why this matters:

1. **GitHub READMEs are crawled and weighted heavily by Google.** A link from
   the README to `hugonissar.github.io/...` is a high-quality inbound link
   from a high-authority domain to your new site. Few signals matter more
   for initial indexing.
2. **AI crawlers cite repo READMEs.** A README that links to a longer-form
   explainer makes the explainer the canonical source for follow-up answers.
3. **Readers who land on the repo first** (from a GitHub search, from a
   Trending list, from a tweet) discover the rest of your work.

Below are drop-in README sections for each of your two repos. Add them near
the top — ideally right after the *"What this is"* section.

---

## For `BigQuery-Read-Only-MCP-Server`

Add this section after *"What this is"*:

```markdown
## 📚 Related writeups

Long-form posts on this project at [hugonissar.github.io](https://hugonissar.github.io):

- [**Self-hosted vs Google's official BigQuery MCP server: a security and cost comparison**](https://hugonissar.github.io/blog/bigquery-mcp-server-comparison/) — side-by-side feature comparison and when to pick each.
- [**Connecting Claude Desktop to BigQuery via MCP in 5 minutes**](https://hugonissar.github.io/blog/claude-desktop-bigquery-mcp/) — the full client-side setup with `mcp-proxy`.
- [**Stopping the $2,000 AI query: how to cap BigQuery scan cost from an MCP server**](https://hugonissar.github.io/blog/bigquery-cost-control-mcp/) — the three layers of cost control with example IAM bindings.

Or read the [project overview](https://hugonissar.github.io/projects/bigquery-readonly-mcp-server/).
```

---

## For `GA4-Ecommerce-BQML-Purchase-Propensity`

Add this section after *"What this is"*:

```markdown
## 📚 Related writeups

Long-form posts on this project at [hugonissar.github.io](https://hugonissar.github.io):

- [**How to build a GA4 purchase propensity model with BigQuery ML (without Vertex AI)**](https://hugonissar.github.io/blog/ga4-purchase-propensity-bqml/) — end-to-end walkthrough of the design.
- [**BigQuery ML vs Vertex AI for marketing ML pipelines: which one and when**](https://hugonissar.github.io/blog/bqml-vs-vertex-ai/) — why BQML beats Vertex for this class of pipeline, with cost numbers.

Or read the [project overview](https://hugonissar.github.io/projects/ga4-ecommerce-bqml-purchase-propensity/).
```

---

## When to add these

**After your first successful site deploy**, not before. The URLs above
won't resolve until the site is live, and the GitHub README preview will
show *"This page isn't working"* in link checkers. Order of operations:

1. Push the site repo.
2. Confirm the workflow ran green and the URL resolves.
3. Open each repo's `README.md` in your editor and paste the relevant
   block from this file near the top.
4. Commit + push. GitHub's README cache typically updates within a minute.

## Optional: a `website` field on each repo

While you're in each repo, go to **About → Edit** (the gear icon next to
the description) and set the **Website** field to `https://hugonissar.github.io`.
This adds a small link at the top of the repo page next to the description,
and (more importantly) it's another high-authority backlink to the site.
