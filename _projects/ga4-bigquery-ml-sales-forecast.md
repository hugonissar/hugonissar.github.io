---
title: "GA4 BigQuery ML Sales Forecast"
tagline: "A pure-SQL pipeline that turns the GA4 BigQuery export into a 30-day daily revenue forecast — with custom holidays, automatic seasonality, ad-spend regressors, and seasonal sub-forecasts for upstream funnel metrics. Output is a single view you connect directly to Data Studio."
repo: GA4-BigQuery-ML-Sales-Forecast
language: SQL
license: MIT
license_url: https://opensource.org/licenses/MIT
status: Pre-release
order: 3
description: "A pure-SQL revenue forecasting pipeline for GA4 ecommerce — ARIMA_PLUS_XREG with custom holidays, ad-spend regressors, and seasonal sub-forecasts on upstream funnel metrics. Trains and runs entirely in BigQuery; output plugs directly into Data Studio. No Python, no Vertex AI, no Cloud Functions."
---

## What this is

A pure-SQL pipeline that produces a **30-day daily revenue forecast** from
your GA4 BigQuery export. Trains an `ARIMA_PLUS_XREG` model on revenue
with four `ARIMA_PLUS` sub-models forecasting the upstream funnel metrics
that feed it. Output is a single BigQuery view designed to plug straight
into Data Studio.

**No Python, no Vertex AI, no Cloud Functions, no orchestrator.** The
whole pipeline is two scheduled BigQuery queries — one that refreshes
data daily, one that retrains weekly — plus a view that re-evaluates on
every query.

## How it works

Two phases:

- **Build** (first run, then weekly retrain) — aggregate GA4 events and
  ad metrics into a daily fact table, then train the main XREG model on
  revenue plus the four univariate `ARIMA_PLUS` sub-models on the funnel
  metrics that become its future regressors.
- **Forecast** (daily refresh, live on every query) — the sub-forecasts
  predict tomorrow's funnel volume; the trained XREG model uses those
  plus your planned ad spend to predict tomorrow's revenue. A view
  unions forecast with history for Data Studio to consume.

After the first run you'll have **11 objects** in your dataset: 3 tables,
6 models, and 2 views — including a `sales_forecast_explained` view
backed by `ML.EXPLAIN_FORECAST` that decomposes every prediction into
trend, seasonality, holiday lift, and per-regressor attribution.

## Why this design

- **Pure SQL.** Lives entirely in BigQuery + scheduled queries. No
  Python orchestrator, no separate ML platform, no IAM service-account
  juggling between systems.
- **Scenario planning is built in.** Because ad spend is an explicit
  regressor with a forecast-time future-values table, you can override
  the trailing-average defaults with a planned media buy and the
  forecast updates the next time the view is queried.
- **Custom occasions, not just calendar holidays.** A `custom_holidays`
  table lets you name your own events — Black Friday week, anniversary
  sales, May campaign — and the model learns a lift per named occasion
  it applies to future occurrences.
- **Explainability as a first-class output.** Every forecasted point
  decomposes into trend, weekly seasonality, yearly seasonality,
  holiday attribution, and per-regressor contribution. If you don't
  know where each predicted dollar comes from, the model isn't ready
  to drive budget decisions.

## A note on model quality

Auto-ARIMA always produces output. That output looks the same whether
the fit is excellent or barely there. Section §4 of the SQL is the
evaluation step that decides whether to trust what you've built — train
an eval model on data ending 30 days ago, run `ML.EVALUATE` on the
held-out window, read the MAPE. Healthy ecommerce MAPE for daily revenue
is **8–15%**; above 20%, investigate before shipping the forecast to a
dashboard.

A longer walkthrough is in the companion post: [Forecasting GA4 revenue
in pure SQL with BigQuery ML — and the model-quality checks you can't
skip](/blog/ga4-bigquery-ml-sales-forecast/).

## Status

This pipeline is **pre-release**. It's been validated on synthetic data
and small production datasets, but real-world ecommerce has quirks that
only show up at scale. I'm actively recruiting testers — see the
[Contributing](https://github.com/{{ site.author.github }}/GA4-BigQuery-ML-Sales-Forecast#contributing)
section of the README for the ideal tester profile. The single most
useful contribution is opening an issue with your MAPE and a sentence
about the business profile, so a public benchmark table can take shape.

## Get it

Full SQL, dataflow diagrams, IAM setup, troubleshooting guide:

[**View on GitHub →**](https://github.com/{{ site.author.github }}/GA4-BigQuery-ML-Sales-Forecast)
