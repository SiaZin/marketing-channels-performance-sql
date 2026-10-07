# SQL-marketing-channels-performance
Marketing channels analysis using SQL in BigQuery: cumulative snapshot deduplication, acquisition funnel metrics, monthly CAC trends, and LTV/CAC comparison across Google, Meta, and TikTok.

## Repository Structure

## Data
Source: SKELAR Data Analytics Intensive (2026)

`marketing_ads_raw` contains raw advertising data from TikTok, Meta, and Google.

| Field | Type | Description |
| --- | --- | --- |
| `source` | STRING | Marketing channel: `tiktok`, `meta`, or `google` |
| `campaign_id` | STRING | Campaign ID |
| `adset_id` | STRING | Ad set ID |
| `ad_id` | STRING | Individual ad ID |
| `date` | DATE | Reporting date |
| `spend` | FLOAT | Cumulative spend in USD at the time of the snapshot |
| `impressions` | INTEGER | Cumulative impressions at the time of the snapshot |
| `clicks` | INTEGER | Cumulative clicks at the time of the snapshot |
| `installs` | INTEGER | Cumulative app installs at the time of the snapshot |
| `registrations` | INTEGER | Cumulative registrations at the time of the snapshot |
| `timestamp` | TIMESTAMP | Time when the ETL process loaded the data (UTC) |

__Cumulative snapshots__

Metrics accumulate within each reporting day, with multiple snapshots recorded for the same ad. Summing all snapshots would overstate the daily totals.

To obtain daily metrics, I retained the latest snapshot by `timestamp` for each `(ad_id, date)` pair. The reporting day is defined by `date`, since a snapshot loaded after midnight can still refer to the previous day - a discovered feature of the dataset.

## Overview

This project analyzes marketing channel performance across Google, Meta, and TikTok using SQL in BigQuery. The goal is to compare the cost of acquiring registered users, examine conversion rates throughout the acquisition funnel, and evaluate acquisition costs against the provided LTV values.

The analysis covers:
1. Preparing cumulative advertising snapshots by retaining the latest record for each ad and reporting day.
2. Aggregating daily performance metrics by marketing channel.
3. Calculating total spend, CPM, CTR, Click → Install and Install → Registration conversion rates, and CAC.
4. Comparing LTV/CAC across channels and examining monthly CAC trends.

The results address four key questions:
-  Which channel has the lowest cost per registered user?
-  Which stage of the acquisition funnel has the lowest conversion rate?
-  Is Meta’s higher advertising spend justified by its acquisition efficiency?
-  How does channel efficiency change over time?

In this project, CAC refers to advertising spend per registered user. LTV values were provided with the dataset.
