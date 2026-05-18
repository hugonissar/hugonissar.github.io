---
title: "Stopping the $2,000 AI query: how to cap BigQuery scan cost from an MCP server"
date: 2026-05-11
last_modified_at: 2026-05-11
repo: BigQuery-Read-Only-MCP-Server
tags: [mcp, bigquery, cost-control, security, ai-agents]
description: >-
  Three layers of cost control for BigQuery queries originating from an MCP
  server — per-query scan ceilings via dry-run, IAM custom roles, and
  project-wide quotas. With code and example IAM bindings.
excerpt: >-
  Every AI agent connected to BigQuery is one bad query away from a
  four-figure invoice. There are three layers of defense; only one of them
  is bulletproof. Here's where to put each.
---

Every AI agent connected to BigQuery is one bad query away from a four-figure
invoice. The agent doesn't have to be malicious — a well-meaning request like
*"find me users with similar behavior to the ones who converted last month"*
can quietly become a `JOIN` of three multi-terabyte tables with no
partition filter. BigQuery happily runs it, scans 6 TB, charges you $30,
and emails the bill.

This post is about preventing that. There are three layers of defense; only
one is bulletproof. The right setup combines all three.

## The three layers

| Layer | Where it lives | What it stops | Bulletproof? |
|---|---|---|---|
| **Application-layer scan ceiling** | Your MCP server (dry-run + reject) | Any single query above the ceiling | Yes — if the ceiling is set |
| **IAM custom role with quota** | GCP IAM (custom role with project quota) | A misconfigured service account exceeding its quota | Mostly — quotas are per-day, not per-query |
| **Project-wide bytes-billed quota** | BigQuery → Quotas | A total spend cap across all queries from the project | Yes — but at project granularity, not user/role |

## Layer 1: application-layer scan ceiling

This is the cheapest and most precise. Every query the MCP server is about
to run gets a **dry-run first**. BigQuery's dry-run returns the estimated
bytes scanned without actually executing the query or charging for it. If
the estimate exceeds your ceiling, the query is rejected before the real
job ever runs.

The Python is short — this is what
[BigQuery-Read-Only-MCP-Server](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server)
does on every query:

```python
from google.cloud import bigquery

bq = bigquery.Client(project=GCP_PROJECT_ID)
MAX_SCAN_MB = int(os.environ.get("MAX_SCAN_MB", "100"))
MAX_SCAN_BYTES = MAX_SCAN_MB * 1024 * 1024

def reject_if_too_big(sql: str) -> None:
    job_config = bigquery.QueryJobConfig(
        dry_run=True,
        use_query_cache=False,
    )
    job = bq.query(sql, job_config=job_config)
    if job.total_bytes_processed > MAX_SCAN_BYTES:
        raise ValueError(
            f"Query would scan {job.total_bytes_processed:,} bytes, "
            f"exceeding cap of {MAX_SCAN_BYTES:,}"
        )
```

**Notes:**

- Dry-runs are free and fast (typically 50–200ms). Caching them with a small
  LRU dramatically reduces overhead when the agent retries the same query.
- `total_bytes_processed` is an estimate, not a guarantee. For highly
  optimized queries against partitioned + clustered tables, the actual scan
  can come in slightly under the estimate. The reverse — actual scan
  exceeding the estimate — is extremely rare in practice, but the defense
  is *belt-and-braces*, so layer 3 still has a role.
- Setting `MAX_SCAN_MB` requires knowing your data. 100 MB is fine for
  exploratory queries against a GA4 export (you can answer most reasonable
  questions in under 100 MB if your tables are partitioned by date). For
  unpartitioned reporting tables, you may need 500 MB. Don't go above 2 GB
  without a very specific reason.

This is the **bulletproof, per-query** defense. No query exceeding the cap
ever runs. No bytes are billed. No surprise charges.

## Layer 2: IAM custom role with quota

The native BigQuery roles (`bigquery.dataViewer`, `bigquery.jobUser`,
`bigquery.user`) don't have per-role byte quotas. You can't say
*"this service account is allowed to scan 50 GB/day"* through them.

You can do it with a **custom role plus a project-level quota override**
keyed to the service account, but the configuration is non-obvious. The
shape:

```bash
# 1. Create the custom role (essentially jobUser + read scoped tighter)
gcloud iam roles create bqAgentJobUser \
  --project=$PROJECT_ID \
  --title="BQ Agent Job User" \
  --permissions=bigquery.jobs.create,bigquery.jobs.get \
  --stage=GA

# 2. Bind to the agent service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:bigquery-readonly-mcp@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="projects/${PROJECT_ID}/roles/bqAgentJobUser"

# 3. Set a per-user-per-day query bytes quota override via the BigQuery
# administration UI: BigQuery → Reservations → Slots/quota → query bytes
# scanned per user per day. Choose a value like 50 GB.
```

The catch: quotas are **per-user-per-day**. They're a backstop, not a
per-query control. A single 6 TB query happily eats through the daily cap
in one shot. So this layer protects against **sustained misbehavior**, not
**a single bad query**.

When this layer pays off: it catches the case where an agent is running 100
well-formed queries an hour for days, slowly burning down your budget.
Layer 1's per-query cap doesn't notice that pattern.

## Layer 3: project-wide bytes-billed quota

This is the **whole project's emergency brake**. In Google Cloud Console:

1. **APIs & Services → Quotas & System Limits**.
2. Filter to BigQuery.
3. Find *"Query usage per day"*.
4. **Edit Quota** and set a daily ceiling. Default is 200 TB/day — drop it
   to something defensible (50 GB? 500 GB?).
5. Save. The change takes effect within minutes.

When this layer matters: a query that somehow slipped past the application
layer ceiling (a bug, a misconfiguration, a deploy without the cap),
combined with an IAM quota that's too loose. The project-wide quota stops
*all* BigQuery jobs in the project when hit — which is disruptive but
bounded. Set it high enough that legitimate batch jobs (your GA4 ML
training, scheduled queries, etc.) don't trip it.

## What this looks like together

For a production deployment of the MCP server I maintain, my actual
configuration:

- **Layer 1**: `MAX_SCAN_MB=100` for exploratory agents, `MAX_SCAN_MB=500`
  for known-good internal use.
- **Layer 2**: a custom role for the service account, with a 50 GB
  per-user-per-day quota.
- **Layer 3**: project-wide cap of 1 TB/day. High enough that nothing
  legitimate ever hits it, low enough that a runaway script gets stopped
  before it costs four figures.

Total cost at idle: zero (Cloud Run scales to zero). Total worst-case
exposure if every layer except the last fails: 1 TB × $6.25/TB = **$6.25**.

## Why "just use IAM" isn't enough

If you ask in a typical Cloud forum *"how do I prevent expensive BigQuery
queries from an agent?"*, the default answer is *"use IAM."* IAM is
necessary. It is not sufficient.

The reason: IAM controls **who can run jobs** and **on which datasets**.
It doesn't control **how big each job is**. A perfectly IAM-correct setup
with `bigquery.dataViewer` on three datasets and `bigquery.jobUser` on the
project lets an agent run a 6 TB scan against the largest of those
datasets, no questions asked. IAM doesn't see the query plan; only
BigQuery does.

The pattern in this post — dry-run before submit, reject above the cap —
is what bridges the gap. IAM gates **access**, the application-layer cap
gates **cost**, the quotas gate **sustained spend**. All three are
necessary, and only the first one is sufficient for stopping a single
catastrophic query.

## FAQ

### Doesn't dry-run also cost money?

No. Dry-runs are explicitly free per
[Google's BigQuery pricing docs](https://cloud.google.com/bigquery/pricing#dry-run).
You can run unlimited dry-runs without billing impact.

### What about queries that scan less than expected?

Dry-run gives an upper bound, not the actual scan. In practice the estimate
is accurate to within a few percent for partitioned + clustered tables, and
within ~10% for unpartitioned ones. Either way, you're being conservative —
you never pay for a query bigger than the dry-run estimate.

### How does this interact with BigQuery's slot-based pricing?

It doesn't directly. The scan ceiling controls **bytes scanned**, which is
the unit of on-demand pricing. If you're on flat-rate slots, the relevant
metric is **slot-seconds**, not bytes — and dry-run reports both. The same
pattern works: dry-run, check slot-seconds, reject if above the cap.

### Can I do this on Snowflake?

Snowflake exposes query plan estimates via `EXPLAIN`. The principle is the
same: estimate before run, reject above a cap. The implementation differs
(Snowflake's estimates are less precise than BigQuery's dry-run) but the
defense-in-depth structure is identical.

## Source code

Full source — including the dry-run cache, the LRU TTL implementation, and
the rest of the security layers (allowlist, rate limiting, result truncation):
[github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server).
MIT-licensed.
