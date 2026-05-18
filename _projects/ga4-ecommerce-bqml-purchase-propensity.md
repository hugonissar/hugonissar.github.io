---
title: "GA4 Ecommerce BQML Purchase Propensity"
tagline: "Self-hosted BigQuery ML pipeline that predicts purchase propensity from GA4 events and pushes the result back to GA4 as a user property via the Measurement Protocol. Built for Google Ads remarketing."
repo: GA4-Ecommerce-BQML-Purchase-Propensity
language: Python
license: MIT
license_url: https://opensource.org/licenses/MIT
status: v0.1.0
order: 2
description: "A complete BigQuery ML purchase propensity pipeline for GA4. Boosted-tree classifier with hyperparameter tuning trains weekly on GA4 export data; a daily Cloud Function scores active users and pushes propensity buckets (low/mid/high) to GA4 via the Measurement Protocol for Google Ads remarketing."
---

## What this is

A complete end-to-end **purchase propensity pipeline** for Google Analytics 4.
A BigQuery ML boosted-tree classifier with hyperparameter tuning trains weekly
on your GA4 export data and predicts which users are most likely to **purchase**
in the next 30 days. A daily Cloud Function (06:00 local time) scores users
active in the last 24 hours and pushes their propensity bucket
(`low` / `mid` / `high`) back to GA4 as a user property via the Measurement
Protocol. GA4 audiences built on that user property sync to Google Ads for
remarketing.

## Use case

> "I want to retarget users who are likely to buy but haven't yet, and stop
> wasting ad spend on recent customers."

Plug it into your GA4 property, point Google Ads at the resulting audiences,
and your `high`-propensity users get a personalized remarketing push within a
day of their last visit. `mid` users get a softer touch. `low` users don't get
spent on.

## Why this design

- **Consent-gated at the push step.** Only users with
  `ads_personalization_consent = 'GRANTED'` are scored and pushed to GA4. The
  filter happens before any propensity data ever reaches Google Ads, so the
  pipeline is compatible with Consent Mode v2 out of the box.
- **Cost-capped.** Every BigQuery query has `maximum_bytes_billed` set
  (default 10 GB). A misconfigured query can't burn money.
- **Recent buyers excluded.** The model doesn't train on or score users who
  bought in the last 30 days. You don't pay Google Ads to retarget converted
  customers.
- **Full GA4 ecommerce funnel.** All 10 standard ecommerce events plus 5
  derived funnel ratios. The strongest predictors —
  `cart_to_checkout_ratio`, `checkout_to_payment_ratio` — are exposed
  to the model.
- **No external dependencies.** No Vertex AI, no AutoML, no third-party SaaS.
  Just BigQuery ML, Cloud Functions, Cloud Scheduler, and the Measurement
  Protocol.

## How is this different from GA4's built-in predictive audiences?

GA4 ships predictive audiences (purchase probability, churn probability) but
requires ≥1,000 returning users with purchases *and* ≥1,000 returning users
without purchases in the past 28 days — a threshold many small and mid-sized
sites don't meet. This pipeline gives you the same capability at lower traffic
levels, with explicit control over features, thresholds, and bucket boundaries.

## Get it

The full source, training SQL, deployment guide, and consent model are on
GitHub. License is MIT.

[**View on GitHub →**](https://github.com/{{ site.author.github }}/GA4-Ecommerce-BQML-Purchase-Propensity)
