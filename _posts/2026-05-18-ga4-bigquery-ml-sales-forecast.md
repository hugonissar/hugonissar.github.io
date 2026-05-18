---
title: "Forecasting GA4 revenue in pure SQL with BigQuery ML — and the model-quality checks you can't skip"
date: 2026-05-18
last_modified_at: 2026-05-18
repo: GA4-BigQuery-ML-Sales-Forecast
tags: [bqml, ga4, forecasting, arima, model-quality]
description: >-
  A pre-release pure-SQL pipeline that turns the GA4 BigQuery export into a
  30-day revenue forecast — with custom holidays, ad-spend regressors, and
  seasonal sub-forecasts. The architecture is interesting; the model-quality
  step is what decides whether to trust the output.
excerpt: >-
  A pre-release SQL-only pipeline produces a 30-day GA4 revenue forecast
  with ad-spend regressors and seasonal sub-forecasts — no Python, no Vertex,
  no Cloud Functions. The architecture matters, but the bigger story is the
  evaluation workflow. Auto-ARIMA produces a confident-looking number whether
  the fit is good or bad. Here's what to check before you trust it.
---

A revenue forecast is a number that looks the same whether the model fit your
data brilliantly or learned almost nothing useful. The chart in Data Studio
draws a line either way. That's the trap with auto-ARIMA forecasting in
BigQuery ML — you get output before you've earned the right to trust it, and
the output is presentable enough to end up in a deck.

This post walks through a new pre-release pipeline I've been building,
[**GA4-BigQuery-ML-Sales-Forecast**](https://github.com/{{ site.author.github }}/GA4-BigQuery-ML-Sales-Forecast),
which produces a 30-day daily revenue forecast in pure SQL from your GA4
BigQuery export. The architecture is interesting — pure SQL, no Python, no
Vertex AI, no Cloud Functions — but the more important story is the
**model-quality checks**, because they're what decides whether the forecast
is useful or decorative.

## TL;DR

| | |
|---|---|
| **What it is** | A pure-SQL pipeline that produces a 30-day daily GA4 revenue forecast, with custom holidays, automatic seasonality, ad-spend regressors, and seasonal sub-forecasts for upstream funnel metrics |
| **How it runs** | One scheduled query refreshes data daily; another retrains the model weekly. Output is a single BigQuery view that plugs straight into Data Studio |
| **Algorithm** | `ARIMA_PLUS_XREG` for the main revenue model, four `ARIMA_PLUS` univariate models for the funnel sub-forecasts |
| **Status** | **Pre-release.** Validated on synthetic and small production datasets. Real-world testing wanted |
| **License** | MIT |
| **The catch** | Auto-ARIMA gives a confident output whether or not the model fits well. Skip §4 of the SQL and you have no idea what you've built |

---

## What the pipeline does

Two phases:

1. **Build** (first run, then weekly retrain). Aggregate GA4 events and ad
   metrics into a daily fact table. Train one `ARIMA_PLUS_XREG` model on
   revenue, plus four small `ARIMA_PLUS` sub-models on the funnel metrics
   that will become the future regressors.
2. **Forecast** (daily refresh, live on every query). The sub-forecasts
   predict tomorrow's funnel volume. The trained XREG model uses those plus
   your planned ad spend to predict tomorrow's revenue. A view unions
   forecast with history for Data Studio.

After running the script you end up with **11 objects** in your dataset —
3 tables, 6 models, 2 views. No Python, no Cloud Functions, no Vertex
endpoints, no orchestrator. Everything is scheduled queries plus views.

### What makes the design interesting

A few choices worth highlighting:

- **Funnel-as-regressors with their own forecasts.** The main model uses
  `view_item`, `add_to_cart`, `begin_checkout`, and ad-spend variables as
  exogenous regressors. The challenge with regressors is always *"how do I
  know tomorrow's value?"* — solved here by training a separate
  `ARIMA_PLUS` model per regressor, so the system forecasts its own inputs.
  Recursive forecasting like this works because the funnel metrics are
  themselves seasonal and stable enough to predict on their own.

- **Custom occasions table.** Built-in holiday regions catch most
  calendar effects, but they don't know about your Black Friday week, your
  May campaign, or your anniversary sale. The `custom_holidays` table lets
  you name these and tag a window around each. The model learns a lift
  per named occasion and applies it to future occurrences.

- **Scenario planning baked in.** Because ad spend is an explicit
  regressor, the forecast reacts to changes in your media plan. Override
  the trailing-average defaults in `future_regressors` with values from a
  `media_plan` table and the forecast updates the next time the view is
  queried.

- **Explainability view as a first-class citizen.** Section §8 of the SQL
  builds a `sales_forecast_explained` view backed by `ML.EXPLAIN_FORECAST`.
  Every forecasted point decomposes into trend, weekly seasonality, yearly
  seasonality, holiday lift, and per-regressor attribution. This is the
  view that makes the eval step actionable rather than abstract.

## Why you cannot trust auto-ARIMA without evaluating it

This is the part that matters most, so it gets its own section.

`ARIMA_PLUS_XREG` is a great default — Google's implementation does
automatic order selection, automatic seasonality detection, automatic
outlier handling, and automatic structural-break detection. The "automatic"
is the appeal and the risk in the same breath. The model **always** fits
something. It always produces a prediction. It always plots a smooth
forecast curve with confidence bands. None of that tells you whether the
fit is any good.

Real ways the same pipeline can go wrong while producing identical-looking
output:

- **The training data has a structural break the model can't reach over.**
  A pricing change six months ago, a new market launched, a viral spike
  during one campaign. Auto-ARIMA may or may not flag these as outliers;
  even when it does, the rest of the model is fit on data that's no longer
  representative of the present. The forecast looks plausible — and is
  systematically biased.
- **A regressor is silently doing nothing.** You added ad spend as a
  regressor expecting it to drive the forecast. The model gave it
  near-zero weight because in your historical data, spend and revenue
  weren't correlated (maybe revenue is dominated by organic, maybe spend
  is too steady to be informative). Now your "spend-aware" forecast isn't
  spend-aware at all, and your scenario plans don't shift the line.
- **Yearly seasonality is over-fit on too little history.** With under
  ~18 months of clean data, the yearly seasonality term picks up noise as
  if it were signal. Next January's forecast reflects last January's
  freak week.
- **A custom occasion was added with one historical example.** The model
  learns whatever happened that one Black Friday and projects it forward
  with high confidence. One data point is not a season.
- **Recent data is missing or stale.** GA4 export delays, a paused tag, a
  broken scheduled query upstream — and the model trains on a truncated
  view of the last 30 days. The trend term picks up the artificial drop
  and forecasts continued decline.

None of those situations produce an error. All of them produce a chart.

## The validation workflow — §4 of the SQL

The repo's section §4 is the eval step, and it's the section that should
never be skipped on a first run or after a major retrain. The pattern:

1. Train an **eval model** identical to production, but on data ending
   **N days ago** (default: 30 days).
2. Run `ML.EVALUATE` against the held-out N days that the eval model
   never saw.
3. Read the metrics.
4. **Drop the eval model.** It exists only for this measurement; the
   production model is the one that trained on everything.

The key SQL for the evaluation step is small enough to read in one breath:

```sql
SELECT *
FROM ML.EVALUATE(
  MODEL `YOUR_PROJECT.YOUR_DATASET.revenue_forecast_eval`,
  (
    SELECT day, revenue, view_item, add_to_cart,
           begin_checkout, spend, impressions, clicks
    FROM `YOUR_PROJECT.YOUR_DATASET.daily_features`
    WHERE day BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
                  AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  ),
  STRUCT(0.95 AS confidence_level, TRUE AS perform_aggregation)
);
```

You get back five numbers: `mean_absolute_error`,
`mean_squared_error`, `root_mean_squared_error`,
`mean_absolute_percentage_error`, and `symmetric_mean_absolute_percentage_error`.

The one you care about most is **MAPE** (mean absolute percentage error).

## What MAPE numbers actually mean

There's a reasonable amount of confusion about what counts as a "good"
MAPE for revenue forecasting because the right answer depends on your
business volatility. Practical operating ranges for daily ecommerce
revenue:

| MAPE | Interpretation | What to do |
|---|---|---|
| **< 8%** | Excellent. Usually means very stable revenue (subscription, B2B) and clean data | Trust it. Use it for planning |
| **8–15%** | Healthy. The pipeline is working as expected for typical ecommerce with normal noise | Trust it. Most forecasts in production live here |
| **15–25%** | Usable but verify. Likely either high day-to-day volatility, modest data history, or a recently-changed business | Use for *directional* planning. Don't pin budget decisions to specific forecast values |
| **25–40%** | Don't trust this in isolation. The model is fitting something but it's noisy enough that a +30% surprise tomorrow wouldn't be unexpected | Investigate before publishing the forecast |
| **> 40%** | Broken. Either data quality, training history, or a structural mismatch with the model | Diagnose; do not deploy |

A subtlety: MAPE penalizes percent errors equally regardless of the day's
volume. A 50% miss on a $100 day and a 50% miss on a $100,000 day both
count the same. If your business has many low-volume days, your MAPE will
read worse than the model deserves — look at **MAE** in actual currency
units to sanity-check.

`SMAPE` (symmetric MAPE) is more forgiving on small denominator days; if
MAPE is much worse than SMAPE on the same period, low-volume days are
distorting the average.

## Reading ML.EXPLAIN_FORECAST: where the prediction comes from

Knowing the model has a decent MAPE is necessary but not sufficient. The
next question is: *where is each forecasted dollar coming from?* If you
deploy a "spend-aware" forecast and 95% of every prediction's value
attributes to the trend term, your forecast isn't actually responding to
spend — it's a stationary line dressed up as a media plan.

The `sales_forecast_explained` view (§8) makes this visible. Each
forecasted day has columns like:

| Column | What it tells you |
|---|---|
| `attribution_trend` | The slow-moving level component |
| `attribution_weekly_seasonality` | The day-of-week effect (your Tuesday is +12%, your Sunday is −20%, etc.) |
| `attribution_yearly_seasonality` | The calendar position effect (you're up 8% relative to year average) |
| `attribution_holiday` | Lift from named occasions (Black Friday, your custom sales) |
| `attribution_spend` | How much of tomorrow's predicted revenue is driven by tomorrow's planned spend |
| `attribution_view_item`, `attribution_add_to_cart`, etc. | Funnel metric contributions |

Two questions to ask every time you look at this view:

1. **Does one term dominate everything else?** A healthy model spreads
   attribution across components. If `attribution_trend` is 90% of every
   forecast, your regressors and seasonality are doing almost nothing —
   the model has effectively collapsed to a simple time-series.

2. **Does the attribution match your intuition?** If you know spend is the
   most important driver of your revenue and `attribution_spend` is near
   zero, something is wrong upstream. Most often: spend and revenue aren't
   actually correlated in your historical data (verify with a quick
   `CORR()`), the regressor was added recently with too little history, or
   the regressor's variance is too low for the model to learn from.

## What to do when the eval fails

Concrete responses to the most common failure modes:

| Symptom | Likely cause | Response |
|---|---|---|
| MAPE > 25% on a long-running stable business | Recent structural break (pricing, market, product mix) | Limit training window to post-break period via `data_window` in the model options. Accept some short-term volatility cost |
| MAPE > 25% on a young business | Not enough history — yearly seasonality overfits | Either disable yearly seasonality (`auto_arima=TRUE` with `seasonal_periods=['WEEKLY']` only) or wait for more history before going to production |
| MAPE looks fine but `attribution_spend` is near zero | Spend and revenue weren't correlated historically | Either remove spend as a regressor, or check whether the right lag is being used (today's spend often doesn't drive today's revenue — try a 1- or 2-day lag) |
| One huge spike day skews everything | Auto-outlier detection didn't catch it | Add the day to `custom_holidays` with `holiday_name='one_off'` and re-train, or filter it out of `daily_features` |
| Forecasts look flat over the next 30 days | Funnel sub-forecasts have too little data to learn weekday seasonality | Confirm `daily_features` has at least 13 months of data and the GA4 ecommerce events are actually firing |

## A note on retraining cadence

The pipeline schedules a weekly retrain by default. That's the right
default for most ecommerce businesses — the underlying behavior changes
slowly enough that weekly is fresh enough, and `ARIMA_PLUS_XREG` training
is cheap enough that running it every Monday morning costs essentially
nothing.

Two cases where you'd want to retrain more often:

- **You're heading into a high-stakes period** (Black Friday, an
  anniversary sale) and the most recent four weeks contain the kind of
  signal you want the model to weight. Daily retrains for two weeks
  before the event, then revert to weekly.
- **A regressor just changed regime** (you doubled your media budget, a
  new channel went live). Retrain on demand once you have 2–3 weeks of
  data at the new level so the model recalibrates its regressor weights.

And one case where you'd want to retrain *less* often:

- **You ran §4 and got a great MAPE.** Don't retrain mid-week if you've
  already validated the current model. Weekly retrains are about keeping
  the model fresh; if the current model is fitting well and the underlying
  business is stable, additional retraining only adds variance.

## This is pre-release — testers wanted

The pipeline has been validated on synthetic data and small production
datasets, but real-world ecommerce data has quirks that only show up at
scale. The README has a [Contributing](https://github.com/{{ site.author.github }}/GA4-BigQuery-ML-Sales-Forecast#contributing)
section laying out exactly what would help most — currently I'm looking
for testers with 2+ years of stable GA4 export history and at least 500
daily orders or 50k daily sessions, especially in less-tested setups:
subscription/recurring revenue, heavy promotional calendars, non-US
holiday regions.

The contribution loop is small: run the pipeline on your data, open an
issue with your MAPE, business profile, and anything that broke or
surprised you. The goal is a public benchmark table of realistic MAPE
numbers across different business types so other users have a calibration
point before they deploy.

## FAQ

### How is this different from Google's GA4 predictive metrics?

GA4 ships predictive metrics (purchase probability, revenue prediction) at
the *user* level — given this user's behavior, how likely are they to
purchase? Useful for audience targeting. This pipeline operates at the
*business* level — given the whole property, what's tomorrow's total
revenue? Different question, different model class, complementary outputs.

### Why ARIMA_PLUS_XREG and not Prophet, NeuralProphet, or DeepAR?

`ARIMA_PLUS_XREG` is the only BQML-native option that supports exogenous
regressors with built-in seasonality and holiday handling. Prophet would
work but requires moving data out of BigQuery into Python. NeuralProphet
and DeepAR require even more infrastructure. For a marketing analytics
team that lives in BigQuery + Data Studio, the operational cost of
introducing a Python orchestrator is high. The pipeline trades off some
modeling ceiling for radically lower ops complexity.

### What if I don't have ad spend data?

You can run the pipeline with only the funnel regressors — drop `spend`,
`impressions`, and `clicks` from §1 and from the model options in §3. The
ad-aware scenario planning goes away, but the seasonal forecasting still
works. Expect MAPE to be 2–5 points worse depending on how much of your
revenue volatility was driven by spend.

### How does this handle Consent Mode v2?

Revenue is computed from `purchase` events in the GA4 export, which
respect consent settings at collection time — non-consented users either
don't appear or appear as modeled estimates depending on your GA4 setup.
The pipeline doesn't add a separate consent filter because at the
aggregate-revenue level you want all observed revenue, modeled or not.

### Can I forecast more than 30 days out?

You can — extend the forecast horizon in `ML.FORECAST` and in
`future_regressors`. Expect accuracy to fall off after roughly 30 days
because the sub-forecasts driving the regressors degrade in the same way
the main model does. For longer horizons, switch to weekly granularity
(aggregate `daily_features` to weekly first) or expect 25–35% MAPE in
weeks 5–8.

### Are there cost limits?

`ARIMA_PLUS_XREG` training scans the full training window each time. For
2 years of daily data with ~10 regressor columns, each train scans on the
order of a few megabytes — well under a cent per train at on-demand
pricing. Daily refresh is similarly trivial. Even on the most generous
weekly retrain + daily refresh schedule, total monthly BigQuery cost from
this pipeline is in the low single dollars.

## Source code

Full SQL, dataflow diagrams, IAM setup, troubleshooting guide, and the
contributing checklist are at
[github.com/{{ site.author.github }}/GA4-BigQuery-ML-Sales-Forecast](https://github.com/{{ site.author.github }}/GA4-BigQuery-ML-Sales-Forecast).
MIT-licensed (placeholder, will finalize for 1.0).

If you run it on production data, please open an issue with your MAPE and
business profile — that's the single most useful contribution at this
stage of the project.
