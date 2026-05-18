---
title: "When +35% is really −60%: a GA4 anomaly detector where the LLM only writes the report"
date: 2026-05-18
last_modified_at: 2026-05-18
repo: GA4-Anomaly-Detector
tags: [mcp, ga4, anomaly-detection, claude, gemini, llm-trust]
description: >-
  A weekend project that started as a benchmark — can an LLM be trusted to
  compute anomalies on raw GA4 data, or should the math stay in code? — and
  became a GA4 anomaly detector with the trust boundary drawn explicitly.
  Plus the full CLI reference and a word of caution about running it as an
  unguarded MCP server.
excerpt: >-
  I've been using MCP servers for about a year, like the rest of the tech
  industry. Plenty of impressive moments. Also plenty of cases where the LLM
  confidently does a calculation wrong on data the SQL clearly returned
  right. This post is about a weekend project that became a tool — a GA4
  anomaly detector where the math runs in code and the LLM only writes the
  prose. Plus the full CLI reference and an honest note on what guardrails
  it doesn't yet have.
---

I've been using MCP servers for about a year now, which means I've been
doing what most of the tech industry has been doing:
hooking AI agents up to real data sources and watching them make us
extremely productive most of the time and occasionally make a confident
mistake that no one without the original SQL output would catch.

The most surprising failure mode, for me, was always the simple
calculations. Not the hard reasoning. Not the complex multi-step joins. The
*simple calculations.* "Revenue grew from $5,000 to $8,000" — fine. "That's
a 60% increase" — sometimes fine, sometimes confidently 35%, sometimes
confidently *negative.* The model returns a number, the chat UI shows the
number in bold, and unless the user happens to do the long division in
their head before pasting it into a Slack channel, the number sticks.

This isn't a controversial observation among people who use these tools
heavily — every analytics engineer I know has stories. But what tends to be
*less* talked about is the architectural lesson: **if the math matters,
don't make the LLM do it.** Hand the LLM the result of the math, computed
deterministically in code, and let it do the part it's actually good at —
writing the prose.

That's the lesson behind a weekend project I shipped this month:
[**GA4-Anomaly-Detector**](https://github.com/{{ site.author.github }}/GA4-Anomaly-Detector).
It started as a benchmark — *can an LLM correctly find anomalies in a
GA4 BigQuery export if I just hand it the rows, or do I need to compute the
anomalies myself first?* — and became a tool that does anomaly detection in
code and uses the LLM purely for narrative. This post walks through what
it does, why the architecture matters, the full CLI reference, and an
honest word of caution about what it deliberately *doesn't* have yet.

## The benchmark that made the design obvious

The original question wasn't even my own. It was already answered in a
[piece by Coupler.io](https://blog.coupler.io/how-to-analyse-ga4-with-ai/)
benchmarking Google's official GA4 MCP server against the GA4 sample
dataset. The setup was simple: ask the agent to describe the recent traffic
trend; check the answer against the actual numbers. **Google's MCP server
reported a 35% increase in traffic when the actual trend was a 60%
decrease.** Same direction missed entirely.

This isn't because Google's engineers built it badly. It's because the
architecture handed the LLM the raw query results and asked it to do the
analysis. That's the structural mistake. An LLM scanning a column of
numbers will sometimes pattern-match to a description that fits the most
salient sub-window rather than the actual trend. Sometimes it will
confidently average the wrong subset of rows. Sometimes it will simply get
the percentage backwards. The chart in your head and the chart in the
LLM's head are not always the same chart.

The fix is structural too: **run the statistics in code, hand the LLM only
the structured findings.** That's what `ga4-anomaly-detector` does. The
LLM never sees the daily metric values. It never sees the raw event rows.
It sees a list of findings ("revenue dropped 38% on 2021-01-19 and stayed
down") and a system prompt forbidding speculation about causes that aren't
corroborated by other findings. If the math is right, the math is right
in code, before any LLM call.

## What it produces

The output is a markdown report you can pipe into Slack, email, a doc, or
`cat`:

```
# GA4 Anomaly Detector
*2021-01-01 → 2021-01-31 · `bigquery-public-data.ga4_obfuscated_sample_ecommerce`*

## Headline
Revenue stepped down by 38% starting 2021-01-19 and has held.

## Key findings
- **revenue** level shift on 2021-01-19: ~$8,200 → ~$5,100
  (↓ -38% sustained over 14 days)
- **conversions** on 2021-01-22: 47 vs ~89 expected
  (↓ -47%, high severity)
- **sessions** on 2021-01-08: 4,820 vs ~3,100 expected
  (↑ +56%, medium severity)

## What changed in the mix
**revenue by source medium** (2021-01-18–2021-01-24 → 2021-01-25–2021-01-31)
- Gainer: `(direct) / (none)` 31% → 44% share
- Loser: `google / cpc` 28% → 17% share
```

The headline at the top is the only sentence the LLM writes that isn't
directly grounded in a numerical finding. Everything below it has a
specific anomaly object with `metric`, `date`, `expected`, `observed`,
`severity` behind it.

## How the math gets done

Three detectors, picked because the alternatives produced worse signal on
GA4-shaped data:

- **Point anomalies use STL residual z-scores.** GA4 metrics have strong
  weekday/weekend cycles. A naive rolling-mean z-score flags every
  Saturday as anomalous. STL decomposes the series into trend + weekly
  seasonal + residual; we z-score the residual. A day is flagged when its
  residual exceeds the sigma threshold.
- **Change points use PELT with an RBF cost.** A one-day spike is not a
  change point. A site migration that drops sessions and they stay dropped
  *is.* PELT separates the two. The RBF cost is roughly scale-invariant,
  so the same penalty works for `sessions` (thousands) and
  `conversion_rate` (single digits).
- **Mix shifts use Jensen-Shannon divergence.** This catches the case
  GA4 dashboards hide: total sessions look flat, but direct doubled while
  organic collapsed. We compute share-of-voice distributions across
  adjacent windows and measure the divergence. JS is bounded `[0, ln 2]`
  so the threshold is interpretable.

The LLM — Gemini 3 by default, but the architecture takes any client
implementing the `LLMClient` Protocol — receives the resulting
`AnomalyReport` and turns it into prose. It cannot invent a finding
because there's nothing in its input it could invent one from.

---

## Full CLI reference

The CLI has two subcommands and a shared set of flags. This section
documents every option in `cli.py`.

### Subcommands

| Command | Purpose |
|---|---|
| `sample` | Run against Google's public obfuscated GA4 sample dataset. No GA4 export of your own needed — only a GCP project for query billing. Sensible defaults for the date range that match the data available in the public sample. |
| `run` | Run against your own GA4 BigQuery export. Requires `--project-id` and `--dataset`. |

### Top-level help flag

| Flag | What it does |
|---|---|
| `-h`, `--help` | Standard argparse help (top level + subcommand names only). |
| `--h`, `--help-all` | Custom action that prints the top-level help *plus every subcommand's full help in one pass*. Use this when you want to see every flag everywhere without typing `-h` per subcommand. The single-dash form `--h` is intentional — `allow_abbrev=False` keeps argparse from auto-expanding it to `--help`. |

### `run`-specific required flags

| Flag | Required | What it is |
|---|---|---|
| `--project-id PROJECT` | Yes | The GCP project hosting your GA4 BigQuery export. |
| `--dataset DATASET` | Yes | The BigQuery dataset, typically `analytics_<property_id>` (e.g. `analytics_123456789`). |

### Date range (both subcommands)

| Flag | Type | Default | What it is |
|---|---|---|---|
| `--start YYYY-MM-DD` | date | `sample`: `2020-12-01` (start of the sample's window). `run`: today minus 30 days. | Inclusive start of the analysis window. |
| `--end YYYY-MM-DD` | date | `sample`: `2021-01-31` (end of the sample's window). `run`: yesterday. | Inclusive end of the analysis window. |

Validation: if `--start > --end`, the CLI exits with code `2` and an error.

### Billing and auth (both subcommands)

| Flag | Default | What it is |
|---|---|---|
| `--billing-project PROJECT` | `GOOGLE_CLOUD_PROJECT` env var | GCP project to bill queries against. **Required**, either via this flag or the env var — the CLI exits with code `2` if neither is set. |

Auth itself is provided by Google's Application Default Credentials —
run `gcloud auth application-default login` once, or set
`GOOGLE_APPLICATION_CREDENTIALS` to a service-account JSON. The CLI
surfaces this hint automatically if BigQuery client initialization fails.

### Data shape (both subcommands)

| Flag | Type | Default | What it is |
|---|---|---|---|
| `--dimensions CSV` | comma-sep list | `source_medium` | Which dimensions to slice mix-shift detection on. Valid values: `source_medium`, `device_category`, `country`, `browser`. Pass `--dimensions ""` (empty string) to skip mix-shift detection entirely — useful on large exports where it's the slowest step. |
| `--metrics CSV` | comma-sep list | All standard metrics (`DEFAULT_METRICS`) | Which metrics to run anomaly detection over. Sticks to the GA4-export-derivable set. |

### Output (both subcommands)

| Flag | Default | What it is |
|---|---|---|
| `-o`, `--output FILE` | stdout | If supplied, the markdown report is written to this path. Otherwise it goes to stdout, ready to pipe into `pbcopy`, `tee`, or anything else. |
| `-v`, `--verbose` | off | Verbose logging on stderr. Logs end up in stderr, output ends up in stdout, so `python cli.py sample > report.md` still works cleanly. |

### Detection tuning (both subcommands, grouped as "Detection tuning")

| Flag | Type | Default | What it is |
|---|---|---|---|
| `--sigma-threshold FLOAT` | float | `3.0` | Z-score threshold for point anomalies. **Lower means more sensitive.** Drop to `2.5` for small-traffic sites with high day-to-day noise; raise to `4.0` for high-volume sites where you only want extreme deviations. |
| `--pelt-penalty FLOAT` | float | `10.0` | Penalty parameter for the PELT change-point detector. **Lower means more change points.** Raise if you're seeing spurious breaks; lower if real shifts are being missed. |
| `--mix-window-days INT` | int | `7` | Width of each comparison window for mix-shift detection. The default compares the last 7 days against the 7 days before, anchored to the most recent date in the export. |
| `--known-events DATES` | comma-sep dates | empty | Dates (YYYY-MM-DD) to exclude from point-anomaly detection. Without this, a December run will dutifully flag Christmas as a -60% anomaly. Excluded dates are also dropped from the noise-floor estimate, so they don't bias the threshold. Example: `--known-events 2026-12-24,2026-12-25,2026-12-31,2027-01-01`. The `holidays` Python package will generate per-country lists if you don't want to hardcode. |
| `--conversion-events CSV` | comma-sep list | `purchase` (the default for ecommerce) | GA4 event names that count as conversions. Matters because the `conversions` metric is derived from this list. SaaS or lead-gen sites should override: `--conversion-events sign_up,subscribe`. Without this, non-ecommerce sites will report 0 conversions and the narrative will confidently say so. |

### LLM narrative (both subcommands, grouped as "LLM narrative")

| Flag | Type | Default | What it is |
|---|---|---|---|
| `--no-llm` | flag | off | Skip the LLM call entirely; use the deterministic template renderer instead. Same output shape, zero tokens, useful for tuning the detector parameters in a tight loop. |
| `--vertex` | flag | off, but reads `GOOGLE_GENAI_USE_VERTEXAI=true` from env | Use Vertex AI (gcloud auth) for the Gemini call instead of an AI Studio API key. Reuses `--billing-project` and your Application Default Credentials. Right choice if you want unified auth + Cloud Logging audit trail; AI Studio + an API key is right for personal use. |
| `--model MODEL` | string | `GeminiClient.DEFAULT_MODEL` (Gemini 3 Preview at the time of writing) | The Gemini model string. Override if you want a different cost/quality tradeoff. |
| `--api-key KEY` | string | `GEMINI_API_KEY`, falling back to `GOOGLE_API_KEY` env var | API key for AI Studio mode only. Ignored in `--vertex` mode. |

### Environment variables the CLI reads

A consolidated list of the env vars the CLI consults, with which flag each
one substitutes for:

| Env var | Substitutes for | Effect |
|---|---|---|
| `GOOGLE_CLOUD_PROJECT` | `--billing-project` | Default project for billing and (in `--vertex` mode) Vertex AI. |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | Path to a service-account JSON, used by Google client libraries' ADC chain. |
| `GOOGLE_GENAI_USE_VERTEXAI` | `--vertex` | If set to `true`, defaults the CLI to Vertex AI mode. |
| `GOOGLE_CLOUD_LOCATION` | — | Vertex AI region. Defaults to `global` (required for Gemini 3 Preview models). Set to e.g. `europe-west4` for data residency, but make sure your model supports that region. |
| `GEMINI_API_KEY` | `--api-key` | Primary fallback for the AI Studio API key. |
| `GOOGLE_API_KEY` | `--api-key` | Secondary fallback if `GEMINI_API_KEY` isn't set. |

### Exit codes

| Code | When |
|---|---|
| `0` | Success — report written or printed. |
| `1` | Operational error — BigQuery auth failed, query failed, or no data was returned for the requested window. The CLI logs an actionable hint (e.g. "run `gcloud auth application-default login`") before exiting. |
| `2` | Usage error — `--start > --end`, an unknown dimension was requested, or no billing project was supplied. |

### Putting it together — three canonical invocations

```bash
# 1. Test against the public sample, no LLM, no tokens spent.
python cli.py sample --billing-project my-gcp --no-llm

# 2. Run against your own export, last two weeks, three dimensions, save to a file.
python cli.py run \
    --project-id my-project \
    --dataset analytics_123456789 \
    --start 2026-05-01 --end 2026-05-15 \
    --dimensions source_medium,device_category,country \
    --output weekly-report.md

# 3. Same, but using Vertex AI for narration (gcloud auth, no API key).
python cli.py run \
    --project-id my-project \
    --dataset analytics_123456789 \
    --start 2026-05-01 --end 2026-05-15 \
    --vertex \
    --model gemini-3-preview \
    --output weekly-report.md
```

---

## Using it from Claude Desktop — and a word of caution

The repo also ships an `mcp_server.py` that exposes the analyze pipeline as
an MCP tool. Any compatible client (Claude Desktop, Cursor, Claude Code,
Gemini CLI) can call it. The server returns the structured findings as
JSON; the client's LLM narrates them in context. This is the closest you
can get to "ask Claude about my GA4 traffic" while keeping the math out of
the LLM's hands.

**But this is the part where I have to be honest about what's missing.**

This repo is **pre-release**. The MCP server in it is intentionally
minimal. It does *not* yet have any of the guardrails my other open-source
MCP server has:

| Guardrail | [BigQuery Read-Only MCP](/projects/bigquery-readonly-mcp-server/) | GA4 Anomaly Detector MCP |
|---|---|---|
| Hard table allowlist | Yes — parsed before query submission | No — it queries whatever dataset you tell it to |
| Per-query scan ceiling | Yes — dry-run + reject above `MAX_SCAN_MB` | No |
| Token-bucket rate limit | Yes — configurable QPM + burst | No |
| Result row cap | Yes — `MAX_RESULT_ROWS` truncation | No |
| API key auth | Yes — constant-time header validation | No — runs locally, trusts the calling client |
| Suitable for multi-tenant or customer-facing deployment | Yes | **No** |

The anomaly detector's MCP server is designed for **single-user, local
use** — it runs on your laptop, against your own GA4 export, called by
your own Claude Desktop. In that context, none of the missing guardrails
matter, because you can't injection-attack your own laptop. But if you're
thinking about deploying this server to Cloud Run or putting it behind a
proxy so a team can share it, **don't**, not without adding the layers
above first.

The longer write-up on why guardrails matter is in
[*Stopping the $2,000 AI query*](/blog/bigquery-cost-control-mcp/) — the
short version is that any MCP server fronted by an LLM is one bad prompt
away from a four-figure invoice unless the *server* refuses to run queries
above a cost cap. The pattern from that post — dry-run, reject above
ceiling — is exactly the patch this repo needs before 1.0.

If you want to use this tool today, the safest mode is the **CLI**, not
the MCP server. The CLI runs once per command, against the dataset you
explicitly point it at, and exits. There's no persistent process for a
prompt injection to talk to.

## FAQ

### Why not just use Looker Studio's built-in anomaly detection?

Looker Studio's anomaly bands are useful for eyeballing. They don't give
you change points, don't give you mix shifts, don't give you a narrative
you can paste into Slack, and don't accept tuning parameters. This tool
produces something closer to what an analyst would write at the end of a
review — *"here are the four things you should know about this week"* —
not a chart you have to interpret.

### Can I plug in a different LLM than Gemini?

Yes. The narrative module defines an `LLMClient` Protocol. `GeminiClient`
implements it; you can drop in an Anthropic or OpenAI client by writing
your own and passing it to `render_with_llm()`. The CLI doesn't have a
flag for that yet — easy patch if anyone wants it.

### Will this work on a small site with low daily traffic?

Probably with tuning. The defaults are picked for medium-traffic ecommerce
(roughly the shape of the public sample dataset). On a small site, drop
`--sigma-threshold` to `2.5` and `--pelt-penalty` to `5.0` so the
detectors actually fire on the smaller signal-to-noise ratio. The mix-shift
detector is more forgiving — it operates on share-of-voice, not raw
volume, so it works fine even at low traffic.

### What about Consent Mode v2 and modeled data?

The detector reads whatever's in your GA4 export. If your export contains
modeled data due to consent mode, the detector treats it the same as
observed data — the math doesn't distinguish. That's usually the right
call for *trend* detection (you want to see the modeled traffic move), but
it means a sudden change in the modeled/observed ratio (e.g., from a
cookie banner change) will show up as a real anomaly. That's correct
behavior; just be aware when interpreting the result.

### Why a weekend project, not a full product?

Because I wanted to know if the architecture worked before committing more
time. The benchmark — *do the structured findings actually make the
narrative more accurate?* — got a clear answer (yes, dramatically) and the
tool was already useful, so I shipped it. The roadmap in the README has
the obvious next steps (synthetic-anomaly tests, more detectors, the
guardrails for MCP mode) — happy to take PRs.

### How does this interact with the BigQuery MCP server?

They're complementary. The
[BigQuery Read-Only MCP server](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server)
is for general-purpose SQL access from an agent, with guardrails. This
anomaly detector is for a specific computation (anomaly detection on GA4)
where the computation is in code and only the narration is the agent's
job. If you connect Claude to *both* servers, you can ask Claude "did
anything weird happen last week" (handled by the anomaly detector) and
then "show me the top 10 affected pages" (handled by the BigQuery server).
Different jobs; different tools.

## Source code

Full source — STL/PELT/JS-divergence detectors, BigQuery fetcher, MCP
server, CLI:
[github.com/{{ site.author.github }}/GA4-Anomaly-Detector](https://github.com/{{ site.author.github }}/GA4-Anomaly-Detector).
MIT-licensed. Pre-release; PRs welcome, especially around the guardrails
work needed before the MCP server is safe to share across a team.
