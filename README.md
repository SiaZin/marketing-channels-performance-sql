# SQL-marketing-channels-performance
Marketing channels analysis using SQL in BigQuery: cumulative snapshot deduplication, acquisition funnel metrics, monthly CAC trends, and LTV/CAC comparison across Google, Meta, and TikTok.

## 🔡Data
Source: SKELAR Data Analytics Intensive (2026)

[`marketing_ads_raw`](marketing_ads_raw.csv) contains raw advertising data from TikTok, Meta, and Google.

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

> [!NOTE]
> [ukr_marketing_channels_performance.pdf](ukr_marketing_channels_performance.pdf) - Full analysis in Ukrainian

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

## Deduplication - daily metrics

To prepare daily metrics, I used two steps:

1. **Select the final cumulative values** for each `(ad_id, date)` pair using `LAST_VALUE()`, ordered by `timestamp`. The explicit window frame includes all snapshots for that reporting day. `SELECT DISTINCT` then reduces the output to one row per ad per day.
2. **Calculate daily increments** by subtracting the previous available reporting date’s cumulative values using `LAG()`. For the first observation of each ad, the previous value is treated as zero.

<details>
<summary>SQL query: preparing daily metrics</summary>
  
```
WITH latest_snapshot AS(
SELECT source,
       campaign_id,
       adset_id,
       ad_id,
       timestamp,
       date,
       spend,
       last_value(spend) over(
          partition by ad_id, date 
          order by timestamp 
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
          ) as day_last_spend,
       impressions,
        last_value(impressions) over(
          partition by ad_id, date 
          order by timestamp 
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
        ) as day_last_impressions,
        clicks,
        last_value(clicks) over(
          partition by ad_id, date 
          order by timestamp 
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
        ) as day_last_clicks,
        installs,
        last_value(installs) over(
          partition by ad_id, date 
          order by timestamp 
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
        ) as day_last_installs,
        registrations,
        last_value(registrations) over(
          partition by ad_id, date 
          order by timestamp 
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
        ) as day_last_registrations
FROM `SQL_homework.marketing_ads_raw` 
ORDER BY ad_id,timestamp
),
one_row_per_day AS (
  SELECT DISTINCT
    source,
    campaign_id,
    adset_id,
    ad_id,
    date,
    day_last_spend,
    day_last_impressions,
    day_last_clicks,
    day_last_installs,
    day_last_registrations
  FROM latest_snapshot
)
SELECT
  source,
  campaign_id,
  adset_id,
  ad_id,
  date,
  ROUND(day_last_spend - coalesce(LAG(day_last_spend) over (partition by ad_id order by date),0),3) AS spend,
  day_last_impressions - coalesce(LAG(day_last_impressions) over (partition by ad_id order by date),0) AS impressions,
  day_last_clicks - coalesce(LAG(day_last_clicks) over (partition by ad_id order by date),0) AS clicks,
  day_last_installs - coalesce(LAG(day_last_installs) over (partition by ad_id order by date),0) AS installs,
  day_last_registrations - coalesce(LAG(day_last_registrations) over (partition by ad_id order by date),0) AS registrations
FROM one_row_per_day
ORDER BY ad_id, date;

-- ---> "marketing_ads_deduplicated" table created
```
</details>

The results were saved as [`marketing_ads_deduplicated.csv`](marketing_ads_deduplicated.csv)

## Daily performance metrics by marketing channel

<details>
<summary>Daily metrics</summary>
  
```
--- денний spend, покази, кліки, встановлення і реєстрації по кожному каналу.
SELECT source, 
       date,
       ROUND(SUM(spend),3) AS spend,
       SUM(impressions) AS impressions,
       SUM(clicks) AS clicks,
       SUM(installs) AS installs,
       SUM(registrations) AS registrations
FROM `SQL_homework.marketing_ads_deduplicated`
GROUP BY source, date
ORDER BY 1,2;

-- -->marketing_ads_channels
```
</details>

The results were saved as [`marketing_ads_channels.csv`](marketing_ads_channels.csv)

## Channel metrics for the entire period

<details>
<summary>Daily metrics</summary>
  
```
SELECT
  source, 
  ROUND(SUM(spend), 3) AS total_spend,
  ROUND(1000 * SUM(spend) / NULLIF(SUM(impressions), 0), 3) AS cpm,
  ROUND(100 * SUM(clicks) / NULLIF(SUM(impressions), 0), 3) AS ctr_pct,
  ROUND(100 * SUM(installs) / NULLIF(SUM(clicks), 0), 3) AS cr_click_install_pct,
  ROUND(100 * SUM(registrations) / NULLIF(SUM(installs), 0), 3) AS cr_install_reg_pct,
  ROUND(SUM(spend) / NULLIF(SUM(registrations), 0), 3) AS cac,
  SUM(registrations) AS total_registrations
FROM `SQL_homework.marketing_ads_channels`
GROUP BY source
ORDER BY source;
```
</details>

## Results
