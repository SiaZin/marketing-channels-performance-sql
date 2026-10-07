# SQL-marketing-channels-performance
Marketing channels analysis using SQL in BigQuery: cumulative snapshot deduplication, acquisition funnel metrics, monthly CAC trends, and LTV/CAC comparison across Google, Meta, and TikTok.

## Repository Structure

## 🔡Data
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

## 📃Overview

This project analyzes marketing channel performance across Google, Meta, and TikTok using SQL in BigQuery. The goal is to compare the cost of acquiring registered users, examine conversion rates throughout the acquisition funnel, and evaluate acquisition costs against the provided LTV values.

The analysis covers:
1. Preparing cumulative advertising snapshots by retaining the latest record for each ad and reporting day.
2. Aggregating daily performance metrics by marketing channel.
3. Calculating total spend, CPM, CTR, Click → Install and Install → Registration conversion rates, and CAC.
4. Comparing LTV/CAC across channels and examining monthly CAC trends. The results address four key questions:
    - Which channel has the lowest cost per registered user?
    - Which stage of the acquisition funnel has the lowest conversion rate?
    - Is Meta’s higher advertising spend justified by its acquisition efficiency?
    - How does channel efficiency change over time?

In this project, CAC refers to advertising spend per registered user. LTV values were provided with the dataset.

## Getting to know the data  

Before preparing the data, I checked for missing and non-positive values, duplicate snapshots, the reporting date range, and the number of snapshots per ad per day.


<details>
<summary>Checking for nulls and negative values</summary>
  
```
SELECT
  COUNTIF(source   IS NULL) AS null_source,
  COUNTIF(campaign_id     IS NULL) AS null_campaign_id,
  COUNTIF(adset_id     IS NULL) AS null_adset_id,
  COUNTIF(ad_id     IS NULL) AS null_ad_id,
  COUNTIF(date    IS NULL) AS null_date,
  COUNTIF(timestamp    IS NULL) AS null_timestamp,  
  COUNTIF(spend    IS NULL) AS null_spend,
  COUNTIF(spend <= 0)       AS zero_or_neg_spend,
  COUNTIF(impressions    IS NULL) AS null_impressions,
  COUNTIF(impressions <= 0)       AS zero_or_neg_impressions,
  COUNTIF(clicks    IS NULL) AS null_clicks,
  COUNTIF(clicks <= 0)       AS zero_or_neg_clicks,
  COUNTIF(installs    IS NULL) AS null_installs,
  COUNTIF(installs <= 0)       AS zero_or_neg_installs,
  COUNTIF(registrations    IS NULL) AS null_registrations,
  COUNTIF(registrations <= 0)       AS zero_or_neg_registrations
FROM `SQL_homework.marketing_ads_raw`;

-- zero_or_neg_clicks,  zero_or_neg_installs, zero_or_neg_registrations мають більше 0
-- Перервіряємо чи немає від'ємних значень
SELECT ad_id, date,
       clicks, installs, registrations
FROM `SQL_homework.marketing_ads_raw`
WHERE clicks <=0 OR installs <= 0 OR registrations <= 0;

-- Все ок.
```
</details>

<details>
<summary>Searching duplicates</summary>
  
```
-- таких дублікатів немає
SELECT ad_id, timestamp, COUNT(*) as cnt
FROM `SQL_homework.marketing_ads_raw`
GROUP BY ad_id, timestamp
HAVING cnt > 1;

--Перевірка на унікальність id(s). 
--adset_id та ad_id мають завжди співпадаючи номера. campaign_id може мати різні adset_id ad_id
SELECT DISTINCT campaign_id||adset_id||ad_id
FROM `SQL_homework.marketing_ads_raw`;
```
</details>

<details>
<summary>Some other checks</summary>
  
```
-- Часовий діапазон 194 дні у 2024 році
SELECT
  MIN(date) AS earliest,
  MAX(date) AS latest,
  DATE_DIFF(MAX(date), MIN(date), DAY) AS days_span
FROM `SQL_homework.marketing_ads_raw`;

-- Для загального розуміння
SELECT ad_id,
       timestamp,
       date,
       count(distinct timestamp) over (partition by ad_id, date) as snapshots_num_total,
       count(timestamp) over (partition by ad_id, date order by timestamp) as snapshots_num_current
FROM `SQL_homework.marketing_ads_raw` 
ORDER BY 1, 2;
```
</details>

> Note: Will be using `date` rather than `DATE(timestamp)`, because for a short time after midnight, the record is still considered part of the previous day in the original table. For instance, a timestamp `'20240104 000659'` corresponds to a date `'20240103'`.

## Deduplication

