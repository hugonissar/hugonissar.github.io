---
title: "How to build a GA4 purchase propensity model with BigQuery ML (without Vertex AI)"
date: 2026-05-15
last_modified_at: 2026-05-15
repo: GA4-Ecommerce-BQML-Purchase-Propensity
tags: [bqml, ga4, bigquery, propensity, google-ads]
description: >-
  A complete walkthrough for building a purchase propensity model on GA4
  ecommerce data using BigQuery ML — boosted trees, hyperparameter tuning,
  consent-aware scoring, and pushing buckets back to GA4 via the Measurement
  Protocol for Google Ads remarketing.
excerpt: >-
  GA4's built-in predictive audiences need ≥1,000 returning purchasers in the
  last 28 days. If you don't have that traffic, you can roll your own with
  BigQuery ML — no Vertex AI, no AutoML, no third-party SaaS. Here's the full
  pipeline.
---

GA4 ships predictive audiences out of the box — *purchase probability*,
*churn probability* — and they're a fine default if you have the traffic for
them. The threshold is ≥1,000 returning users with purchases *and* ≥1,000
returning users without, in the past 28 days. Many small and mid-sized
ecommerce sites never cross it.

If that's you, you can roll your own with **BigQuery ML**. No Vertex AI, no
AutoML, no third-party SaaS — just BigQuery, a Cloud Function, the GA4
Measurement Protocol, and Cloud Scheduler. This post walks through the design
end-to-end, with the full source open on GitHub.

## What the pipeline looks like

```
GA4 → BigQuery export → weekly BQML training → daily Cloud Function scoring
                                                          ↓
                                            GA4 Measurement Protocol push
                                                          ↓
                                            GA4 audiences → Google Ads
```

Two files, one model, one Cloud Function. Weekly training runs as a scheduled
BigQuery query (Sunday 00:01). Daily scoring runs as a Cloud Function (06:00
local time) that scores every user active in the last 24 hours, buckets them
into `low` / `mid` / `high`, and pushes the bucket back to GA4 as a user
property via the Measurement Protocol.

## Why BQML over Vertex AI for this

For a single binary classification model on GA4 data, BQML is the right tool.
Three reasons:

1. **Data stays in BigQuery.** Training reads directly from the GA4 export.
   No data movement, no scheduling, no separate feature store.
2. **HP tuning is built in.** `OPTIONS(num_trials=20, ...)` sweeps
   `learn_rate`, `max_tree_depth`, `subsample`, and `l2_reg` and reports
   `ML.TRIAL_INFO` after training. No separate orchestration.
3. **Inference is a SQL query.** `ML.PREDICT(MODEL ...)` returns scored rows.
   The whole daily job is one BigQuery query plus an HTTP loop.

Vertex AI is the right answer when you outgrow this — multi-model serving,
custom containers, GPU training. For a ~30-feature boosted-tree classifier on
180 days of GA4 events, BQML is simpler, cheaper, and equally accurate.

## The features

The model uses ~30 features across five categories:

| Category | Features |
|---|---|
| Engagement | `total_engagement_seconds`, `engagement_last_30d_seconds`, `engagement_last_7d_seconds` |
| Sessions | `sessions_total`, `engaged_sessions`, `engaged_session_rate`, `avg_session_depth`, `avg_session_engagement_seconds`, `max_session_engagement_seconds` |
| Funnel events | `view_item_list_count`, `select_item_count`, `view_item_count`, `add_to_wishlist_count`, `add_to_cart_count`, `view_cart_count`, `remove_from_cart_count`, `begin_checkout_count`, `add_shipping_info_count`, `add_payment_info_count` |
| Activity | `page_view_count`, `total_events`, `days_since_last_visit` |
| Funnel ratios | `list_to_view_ratio`, `view_to_cart_ratio`, `cart_to_checkout_ratio`, `checkout_to_payment_ratio`, `cart_abandon_ratio` |

The ratios matter most. In every ecommerce model I've trained, the top three
feature importances are some combination of `checkout_to_payment_ratio`,
`add_payment_info_count`, and `cart_to_checkout_ratio`. Users who reach
checkout but don't pay are the highest-value retargeting segment.

## Aren't `begin_checkout` and `add_payment_info` data leakage?

A common worry. The short answer: no, they aren't.

Data leakage means using post-prediction-time information at training time.
This model trains features from the **180-day feature window** (210 to 31
days ago) and labels from a **separate 30-day window** (30 to 1 day ago). The
windows don't overlap. An `add_payment_info` event 60 days ago is a real,
legitimate predictor of a `purchase` event 10 days ago — exactly the signal
you want the model to learn.

The full training SQL with the windowing is in `training.sql`
in the [repo](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity).

## Consent: filter at the push step, not at training

This is the design decision that gets the most pushback, so it's worth being
explicit.

**Training data is not filtered by consent.** **Scoring data is.**

The pipeline reads all GA4 events at training time to maximize the positive
class. A high-intent user behaves the same regardless of which consent button
they clicked, so filtering training data to consented users only shrinks the
positive set by 30–40% (typical EEA rates) and materially hurts model quality.

At the **scoring step**, the SQL filters to
`ads_personalization_consent = 'GRANTED'`. Non-consented users never get a
propensity bucket, never enter the GA4 audience, never reach Google Ads. The
moment data is *used for ad targeting* is the moment consent legally binds —
that's the push step, and that's where the filter lives.

If your organisation requires a stricter "no non-consented data in any
downstream system" policy, the repo has a one-CTE patch that adds a
consent filter to training too. See
[the README's Consent model section](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity#-consent-model).

## Three buckets, fixed thresholds

The model outputs a probability. The pipeline buckets it:

- `low` — predicted probability < 0.40
- `mid` — 0.40 ≤ p < 0.70
- `high` — p ≥ 0.70

Thresholds are **fixed**, not percentile-based, so bucket meaning stays stable
across retrains. The first model run gives you a distribution; revisit the
thresholds once if it's heavily skewed. After that, leave them alone.

## How much traffic do you need?

Realistic operating points, assuming a 3% conversion rate:

- **Workable:** 1,000+ weekly visitors / 30+ weekly purchases — usable for
  ranking, expect noise in bucket sizes.
- **Comfortable:** ~3,300 weekly visitors / ~100 weekly purchases — stable AUC.
- **Ideal:** ~8,000 weekly visitors / ~250 weekly purchases — reliable for production.
- **Robust:** ~16,000 weekly visitors / ~500 weekly purchases — safe to
  fully automate against ad spend.

Below 1,000 weekly visitors, both this pipeline and GA4's built-in audiences
struggle. The issue is statistical, not technical — there just aren't enough
positives to train against.

## Deploying it

The repo has a six-step quick start: create the BQML dataset, train the model
once manually, schedule the weekly training as a BigQuery scheduled query,
deploy the Cloud Function, schedule the daily run with Cloud Scheduler, and
verify in GA4 DebugView. Total setup time is about an hour if you have GA4
BigQuery export already turned on; longer if you don't (you need both daily
and streaming exports on).

The full quick-start is in the
[repository README](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity#-quick-start).

## FAQ

### Why not write audiences directly via the Google Ads API?

The Ads API audience push requires more permissions, more setup, and doesn't
keep GA4 reporting in sync. Writing back as a GA4 user property keeps
everything in one place: GA4 reports, GA4 audiences, and Google Ads remarketing
all read from the same source.

### Why exclude recent buyers from both training and scoring?

Two reasons. First, retargeting people who just bought wastes ad spend.
Second, in training, frequent buyers dominate the positive class — the model
learns *"frequent buyers buy again"* instead of *"high-intent prospects buy."*
You don't need ML to target recent buyers; create a rule-based audience in GA4
(`purchase event in last 30 days`) and run retention campaigns against it
separately.

### Can I run this without Consent Mode v2?

Technically yes (remove the consent filter from the SQL), but then you can't
legally push the data to Google Ads for ad targeting in jurisdictions that
require explicit consent. The pipeline is designed consent-first because
that's the constraint that actually matters in production.

### Does it work with Firebase / app properties?

With minor changes to the SQL — `user_pseudo_id` exists in app exports too,
but the consent param keys differ. Easy to adapt.

## Source code

Full source, training SQL, IAM least-privilege guide, and Cloud Function
deployment script on GitHub:
[github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity).
MIT-licensed.
