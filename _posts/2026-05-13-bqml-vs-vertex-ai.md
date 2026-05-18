---
title: "BigQuery ML vs Vertex AI for marketing ML pipelines: which one and when"
date: 2026-05-13
last_modified_at: 2026-05-13
repo: GA4-Ecommerce-BQML-Purchase-Propensity
tags: [bqml, vertex-ai, ga4, ml]
description: >-
  A practical comparison of BigQuery ML and Vertex AI for marketing ML
  pipelines — purchase propensity, churn, LTV. Where each one wins, the
  threshold at which to migrate, and why most teams start in the wrong place.
excerpt: >-
  Most marketing ML pipelines start in the wrong place. BQML is the better
  default for single-model jobs against GA4 data; Vertex AI wins for
  multi-model serving, custom containers, and anything that needs a feature
  store. Here's the decision framework.
---

Marketing teams new to ML on Google Cloud tend to start in the wrong place.
The default suggestion in most blog posts and the Vertex AI marketing
material is **Vertex AI** — it's the flagship, it has a UI, and it bundles
AutoML. But for the actual job most marketing teams need done — train one
classifier on GA4 data, score users daily, push the result somewhere —
**BigQuery ML** is the better tool.

This post is the decision framework. By the end you'll know which one to
pick and the specific threshold at which it's worth migrating from one to
the other.

## TL;DR

| Question | Use BQML | Use Vertex AI |
|---|---|---|
| One model, trained periodically | ✓ | |
| Multiple models for the same use case | | ✓ |
| Data already lives in BigQuery | ✓ | |
| Need custom Python ML libraries (PyTorch, JAX, custom XGBoost) | | ✓ |
| Need a feature store | | ✓ |
| Need GPU training | | ✓ |
| Need explainability via SHAP/IG | | ✓ (BQML has `ML.GLOBAL_EXPLAIN` but it's lighter) |
| Inference is a SQL query | ✓ | |
| Inference needs sub-100ms latency | | ✓ (Vertex AI endpoints) |
| Inference is daily batch | ✓ | |
| Total annual MLOps budget is "as little as possible" | ✓ | |

For a purchase propensity model trained on GA4 export data with daily batch
scoring, **BQML wins on every relevant axis**.

## Where the two actually differ

The marketing comparison material tends to overplay one side or the other.
The real differences worth caring about:

### 1. Where the data lives

BQML trains *inside* BigQuery. Your training data is a `SELECT` statement;
the model is a SQL `CREATE MODEL` statement; inference is `ML.PREDICT(MODEL ..., (SELECT ...))`.
Nothing moves. You never write to GCS, you never spin up a Vertex training
job, you never wait for a container to pull.

Vertex AI assumes data needs to move into a training pipeline. For GA4
data that's already in BigQuery, that's friction with no benefit. For
data scattered across multiple databases or sitting in raw form on GCS,
Vertex AI's pipelines pay for themselves.

### 2. Algorithm choice

BQML's algorithm catalog is narrow but deep:

- Linear and logistic regression
- Boosted trees (XGBoost under the hood) — the workhorse for most marketing ML
- Random forest
- DNN (small)
- k-means
- Matrix factorisation
- ARIMA / ARIMA_PLUS for forecasting
- AutoML Tables (via Vertex)
- Imported TensorFlow / ONNX models

For purchase propensity, churn, or LTV regression, **boosted trees are the
right algorithm** in 9 out of 10 cases. BQML's `BOOSTED_TREE_CLASSIFIER` with
hyperparameter tuning produces results within 1–2% AUC of the best you'll
get from Vertex AI's AutoML Tables for the same data — and you get the model
in 20 minutes instead of 8 hours.

Vertex AI wins outright if you need:
- Transformer models for text
- Computer vision
- Custom training code with PyTorch/JAX
- Anything Vertex AI Model Garden offers that BQML doesn't

### 3. Hyperparameter tuning

Both support HP tuning. BQML does it via `OPTIONS(num_trials=20, ...)` in
the `CREATE MODEL` statement. Vertex AI does it via Vizier-backed pipelines.

In practice, BQML's HP tuning is enough for marketing models. The search
spaces that matter — learning rate, tree depth, subsample ratio, L2
regularization — are all exposed and sweepable. Vizier shines when the
search space is high-dimensional and the objective surface is jagged. For
GBM-class models on tabular marketing data, neither of those is true.

### 4. Serving

This is the dimension where Vertex AI wins clearly when it matters.

- **BQML serving = run `ML.PREDICT` as a SQL query.** Latency is BigQuery
  latency — typically 2–5 seconds. Fine for batch, fine for daily, fine
  for "user opens a dashboard and sees a score." Not fine for "user clicks
  buy and we need to decide if we offer a discount in 50ms."

- **Vertex AI serving = deploy a model to an endpoint.** Sub-100ms latency,
  autoscaling, traffic splitting, blue-green deployments. The MLOps stuff
  matters here.

If your model output drives a daily Google Ads audience refresh, BQML's
latency profile is more than fine. If it drives on-site personalisation in
real time, you need Vertex AI.

### 5. Cost

This is the dimension where BQML's lead is biggest and least talked about.

**BQML costs:**
- Training: priced as a BigQuery query, charged per byte scanned during
  training. A weekly retrain over 180 days of GA4 data for a mid-sized
  ecommerce site is typically 5–20 GB scanned, costing under $0.50 per train.
- Inference: priced as a BigQuery query. Same scan-based pricing. A daily
  scoring run is usually well under $1.

**Vertex AI costs (managed endpoint, n1-standard-4):**
- Training: the same, basically — Vertex pulls from BigQuery for tabular models.
- **Serving: ~$0.20/hour per node, 24/7 = ~$146/month minimum.** This is the
  killer for marketing teams running batch workloads. You're paying for
  serving capacity you don't use.

For a typical marketing ML setup running daily batch jobs, BQML is somewhere
between **15× and 100× cheaper** at steady state than Vertex AI.

You can mitigate this on Vertex by using batch prediction jobs instead of
endpoints — those are charged per prediction. But then you've given up the
main thing Vertex AI brought to the table (online serving) and added
orchestration complexity.

## The threshold at which to migrate

A clear-cut migration trigger isn't *"my model got bigger"* — it's a change
in how the model gets used. Specifically:

| Trigger | Migration needed? |
|---|---|
| You need sub-100ms inference (on-site personalization, real-time bidding) | **Yes, migrate to Vertex AI endpoints.** |
| You need more than one model for the same use case (per-region, per-vertical) | **Yes** — BQML can do this but Vertex AI's model registry is built for it. |
| You need a feature store (features reused across many models) | **Yes** — Vertex AI Feature Store is the right tool. |
| You're using deep learning architectures BQML doesn't ship | **Yes.** |
| Your model is slow to train (>2 hours) and you need GPU | **Yes.** |
| Your data outgrew BigQuery's free training budget | **Maybe**, more often the right answer is feature engineering or sampling. |
| Your model got bigger and slower | **No** — usually the answer is fewer features or column-pruned views. |

Most marketing teams never hit any of these triggers. They stay on BQML
indefinitely, retrain weekly, score daily, and the system runs for years
with effectively zero MLOps overhead.

## A note on AutoML Tables

AutoML Tables (now part of Vertex AI) is the closest direct competitor to
BQML's `BOOSTED_TREE_CLASSIFIER` for tabular data. It builds an ensemble
under the hood and routinely produces models 1–3% AUC better than a
hand-tuned single GBM.

Worth it? Sometimes. AutoML Tables takes 6–10 hours to train (vs. 20
minutes for BQML) and serves out of a Vertex endpoint (so the cost story
above applies). For most marketing models that 1–3% AUC translates to a
single-digit-percent improvement in target audience precision, which is
meaningful but often not worth the cost and complexity.

The exception: **rare-event prediction.** If your conversion rate is below
0.5% and you're working at meaningful scale, AutoML Tables' ensemble usually
beats BQML noticeably enough to be worth it. Below 100k positive examples,
the gap is much smaller.

## What I actually use

For the [GA4 purchase propensity pipeline](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity)
I maintain, the answer is BQML, end to end:

- `BOOSTED_TREE_CLASSIFIER` with `num_trials=20` HP tuning.
- Trained weekly as a scheduled BigQuery query.
- Scored daily inside a Cloud Function that calls `ML.PREDICT` once and
  pushes the results to GA4 via the Measurement Protocol.
- Total monthly cost: roughly $2–5 for a mid-sized ecommerce site.

The repo has the full training SQL, the Cloud Function source, IAM
least-privilege setup, and a complete deployment guide.

## FAQ

### Can I export a BQML model to Vertex AI later?

Yes for some model types. `BOOSTED_TREE_CLASSIFIER` exports as a TensorFlow
SavedModel via `EXPORT MODEL`, which can be imported into Vertex AI's Model
Registry. Useful escape hatch if you outgrow batch serving and need an
online endpoint later — you don't have to rebuild.

### Does BQML support model monitoring?

Lightly. `ML.EVALUATE` gives you per-evaluation metrics, and you can
schedule it as a query that writes to a monitoring table. You don't get
Vertex AI's drift detection or alerts out of the box. For most marketing
models, scheduled `ML.EVALUATE` + a Cloud Monitoring log-based alert on
ROC AUC dropping is enough.

### Are there feature importance / explainability tools in BQML?

Yes — `ML.GLOBAL_EXPLAIN` gives you global feature importance, and
`ML.EXPLAIN_PREDICT` gives per-prediction Shapley-style attributions. Less
flexible than Vertex AI's full SHAP/IG suite, but enough for the questions
marketing teams typically ask.

### Why not Snowflake / Databricks ML for the same use case?

The same tradeoff replicated in a different ecosystem. If your data is in
Snowflake, Snowflake's ML functions are the BQML equivalent. The decision
framework above applies the same way — start in the warehouse, migrate to a
managed ML platform only when a real serving or modeling need forces it.

## Source code

The full BQML pipeline I described — training SQL, Cloud Function, IAM
setup — is at
[github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity).
MIT-licensed.
