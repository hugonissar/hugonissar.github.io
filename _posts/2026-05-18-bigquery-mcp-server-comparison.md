---
title: "Self-hosted vs Google's official BigQuery MCP server: a security and cost comparison"
date: 2026-05-18
last_modified_at: 2026-05-18
repo: BigQuery-Read-Only-MCP-Server
tags: [mcp, bigquery, cloud-run, security, ai-agents]
description: >-
  When to pick Google's managed BigQuery MCP server vs a self-hosted one. A
  side-by-side comparison of allowlists, scan ceilings, rate limiting, and idle
  cost — with deploy commands for the self-hosted alternative.
excerpt: >-
  Google's official BigQuery MCP server is the right default for trusted
  internal analysts. For agents handling untrusted input, a self-hosted server
  with hard table allowlists and per-query scan ceilings is the safer choice.
  Here's the side-by-side.
---

Google released an official **BigQuery MCP server** in 2025. It works, it's
maintained, it's the right default for many teams. But the moment you put an
agent in front of an untrusted user — a customer chatbot, a third-party
integration, anywhere prompt injection is a real threat — its defaults stop
being the right defaults.

This post compares the two architectures across the dimensions that actually
matter in production: **table access control**, **per-query scan cost**,
**rate limiting**, and **idle cost**. By the end you'll know which one to
deploy and why.

## TL;DR

| | Google official | Self-hosted (this guide) |
|---|---|---|
| Table access control | IAM only — every reachable table | Hard allowlist of `(dataset, table)` pairs, parsed in code |
| Per-query scan cap | None built in | `MAX_SCAN_MB` enforced via dry-run before the job runs |
| Rate limiting | None ("no limit on the number of calls", per Google's docs) | Token bucket + burst, configurable |
| Result row cap | None | `MAX_RESULT_ROWS`, truncates server-side |
| Source code | Closed | Open, MIT, ~1400 lines |
| Idle cost | Managed | ~$0 (Cloud Run scales to zero) |
| Prompt-injection scanning | Yes (Model Armor add-on) | No |
| Forecasting built in | Yes | No |

**Pick Google's server** if you trust the agent absolutely, want Model Armor
prompt-injection scanning, or need built-in `forecast` and ARIMA tools.

**Pick the self-hosted alternative** for anything else — especially customer-facing
agents, multi-tenant deployments, regulated environments, and anywhere the
words *"scan budget"* or *"rate limit"* matter.

## The three risks Google's defaults don't cover

### 1. Any table the service account can reach

Google's server exposes every BigQuery table the underlying service account
has IAM permission to read. For an internal data team that's the right default
— the service account has the same scope as the analyst using it.

For an agent talking to a customer, it isn't. A misconfigured IAM grant, a
fork of the service account's role, or a future colleague who adds a dataset
to *"the analytics service account, it's already got access to everything"*
can quietly widen the surface that an agent can read. The next prompt
injection now has more to work with.

The self-hosted alternative inverts the default: tables are listed explicitly
in `(dataset, table)` pairs as environment variables. Anything outside the
allowlist is rejected at the **SQL parser layer** — before a job is ever
submitted to BigQuery. Cross-dataset references like
`wrong_dataset.allowed_table` are caught and rejected too.

### 2. The "AI just ran a $2,000 query" problem

There is no per-query scan ceiling in Google's MCP server. The
[official docs](https://docs.cloud.google.com/bigquery/docs/use-bigquery-mcp)
suggest enforcing one via custom IAM roles or BigQuery-level quotas. Both work,
both require Ops effort, and neither is on by default.

The self-hosted server dry-runs every query first. If BigQuery's estimate
exceeds `MAX_SCAN_MB` (default 100), the real job is never submitted — no
bytes billed, no surprise on the invoice.

### 3. No built-in rate limiting

From Google's docs, verbatim: *"The BigQuery MCP server doesn't have its own
quotas. There is no limit on the number of calls that can be made to the MCP
server."*

For a server fronted by an agent that retries on failure, that's a foot-gun.
The self-hosted alternative ships with a configurable
`RATE_LIMIT_QPM` and `RATE_LIMIT_BURST` token bucket, plus separate
concurrency semaphores for queries and metadata calls.

## When the managed version still wins

The self-hosted server isn't a strict superset. Google's managed offering
gives you **Model Armor integration** for prompt-injection scanning (a paid
add-on, but a real one) and **built-in forecasting / ARIMA tools** that the
self-hosted server intentionally doesn't ship. If those matter, use Google's
version. If they don't, the cost / security tradeoff swings hard toward
self-hosted.

## Deploying the self-hosted alternative

Five gcloud commands, ~10 minutes:

```bash
# 1. Generate API keys
openssl rand -hex 32 | gcloud secrets create mcp-api-key --data-file=-
openssl rand -hex 32 | gcloud secrets create mcp-admin-key --data-file=-

# 2. Build the image
gcloud builds submit \
  --tag europe-north2-docker.pkg.dev/$PROJECT_ID/bigquery-readonly-mcp/server:latest

# 3. Deploy to Cloud Run
gcloud run deploy bigquery-readonly-mcp \
  --image=europe-north2-docker.pkg.dev/$PROJECT_ID/bigquery-readonly-mcp/server:latest \
  --region=europe-north2 \
  --service-account="bigquery-readonly-mcp@$PROJECT_ID.iam.gserviceaccount.com" \
  --set-secrets="MCP_API_KEY=mcp-api-key:latest,MCP_ADMIN_KEY=mcp-admin-key:latest" \
  --set-env-vars="GCP_PROJECT_ID=$PROJECT_ID,BQ_DATASET_ID=analytics,BQ_ALLOWED_TABLE=events,MAX_SCAN_MB=100,RATE_LIMIT_QPM=20"
```

The full IAM least-privilege guide, multi-table configuration, and operations
playbook are in the [repository README](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server).

## Frequently asked questions

### Does this work with Claude Desktop?

Yes. It also works with Cursor, Windsurf, Claude Code, ChatGPT's deep
research, the OpenAI Responses API, and anything else that speaks
streamable-HTTP MCP. Configuration is the standard `url` + `headers` MCP
client config.

### Can I run it inside a private VPC?

Yes. Add `--vpc-connector` and `--ingress=internal` to the Cloud Run deploy
command, then put an internal load balancer in front.

### How much does it cost at idle?

Roughly nothing. Cloud Run scales to zero between requests; the only standing
costs are Artifact Registry storage (~$0.10/GB/month for the image) and
Secret Manager versions (cents/month). Under modest load — say 1000 queries
a day — total monthly cost is in the single-digit dollars.

### What about column-level masking?

The allowlist is table-granularity. If you need to hide PII columns, create
a BigQuery authorized view that projects only safe columns and add the *view*
to the allowlist instead of the underlying table.

## Source code

The full source is one Python file, MIT-licensed, ~1400 lines:
[github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server).
PRs welcome.
