# kpi-metrics-dashboard

# E-Commerce Customer Journey & Conversion Dashboard – README

This dashboard visualizes user behavior and e-commerce performance for the Google Merchandise Store (August 2016). It consists of two pages: a high-level **Executive Overview** and a detailed **Funnel Analysis**, both segmented by device (desktop, mobile, tablet). The dashboard is fully interactive with a shared device filter.
The dashboard is fully interactive with a shared device filter and is powered entirely by custom SQL queries on BigQuery public datasets.

🔗 **Live Dashboard**: [Customer Journey & Conversion Dashboard (Looker Studio)](https://lookerstudio.google.com/reporting/5ba94044-8343-40f2-bf4c-c7770eab143d)

---

## Page 1: Executive Overview: KPIs

This page presents high-level business metrics to give a snapshot of user and revenue activity.

### Key Visuals:
- **Scorecards**:
  - Sessions
  - Transactions
  - Revenue (USD)
  - Conversion Rate
  - Average Order Value

- **Time Series Graphs**:
  - Revenue, Transactions, Conversion Rate over time
  - Includes optional metric switcher and dropdown

- **Geographic Map**:
  - Transactions by country

### Device Filter:
- All components respond to a dropdown that filters by device type.

---

## Page 2: Funnel Analysis

This page visualizes the step-by-step conversion journey.

### Components:
1. **Funnel Chart** (Sessions → Product Views → Add to Cart → Checkout → Transactions)
2. **Mini Scorecards** for each step (with optional sparklines)

### Device Filter:
- Shared dropdown allows comparison across desktop, mobile, and tablet users

---

## Data Sources 

Both dashboard pages are powered by custom SQL queries run directly on the **Google Analytics 360 BigQuery sample dataset**:

- **Dataset Name**: `bigquery-public-data.google_analytics_sample`
- **Table Pattern**: `ga_sessions_*`
- **Reference**:
  - [Google Analytics Sample Dataset – BigQuery Public Datasets](https://cloud.google.com/bigquery/public-data)
  - [GA360 sample (Google Support)](https://support.google.com/analytics/answer/7586738?hl=en)

## BigQuery Custom Queries

### 1. KPI metrics
> Used for the KPI overview (first page) 

```sql
SELECT
  PARSE_DATE('%Y%m%d', date) AS session_date,
  channelGrouping AS channel,
  trafficSource.source AS source,
  trafficSource.medium AS medium,
  device.deviceCategory AS device,
  geoNetwork.country AS country,
  SUM(IFNULL(totals.visits, 0)) AS sessions,
  SUM(IFNULL(totals.transactions, 0)) AS transactions,
  ROUND(SUM(IFNULL(totals.transactionRevenue, 0)) / 1000000, 2) AS revenue_usd,
  SAFE_DIVIDE(
    SUM(IFNULL(totals.transactions, 0)),
    SUM(IFNULL(totals.visits, 0))
  ) AS conversion_rate,
  IFNULL(
    SAFE_DIVIDE(
      ROUND(SUM(IFNULL(totals.transactionRevenue, 0)) / 1000000, 2),
      NULLIF(SUM(IFNULL(totals.transactions, 0)), 0)
    ), 0
  ) AS avg_order_value
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_*`
WHERE
  _TABLE_SUFFIX BETWEEN '20160801' AND '20160831'
GROUP BY
  session_date, channel, source, medium, device, country
ORDER BY
  session_date
```

### 2. Funnel Table (Long Format)
> Used for the funnel chart – output reshaped using UNION ALL to provide step names as row values.

```sql
WITH base AS (
  SELECT
    CONCAT(fullVisitorId, CAST(visitId AS STRING)) AS session_id,
    device.deviceCategory AS device,
    totals.transactions AS transactions,
    hits.eCommerceAction.action_type AS action_type,
    hits.type AS hit_type,
    product.v2ProductName AS product_name
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
    UNNEST(hits) AS hits
    LEFT JOIN UNNEST(hits.product) AS product
  WHERE
    _TABLE_SUFFIX BETWEEN '20160801' AND '20160831'
),

aggregated AS (
  SELECT
    device,
    COUNT(DISTINCT session_id) AS sessions,
    COUNT(DISTINCT IF(hit_type IN ('PAGE', 'EVENT') AND product_name IS NOT NULL, session_id, NULL)) AS product_views,
    COUNT(DISTINCT IF(CAST(action_type AS INT64) = 2, session_id, NULL)) AS add_to_cart,
    COUNT(DISTINCT IF(CAST(action_type AS INT64) = 3, session_id, NULL)) AS checkout,
    COUNT(DISTINCT IF(transactions >= 1, session_id, NULL)) AS transactions
  FROM base
  GROUP BY device
)

SELECT device, 'Sessions' AS funnel_step, sessions AS value FROM aggregated
UNION ALL
SELECT device, 'Product Views', product_views FROM aggregated
UNION ALL
SELECT device, 'Add to Cart', add_to_cart FROM aggregated
UNION ALL
SELECT device, 'Checkout', checkout FROM aggregated
UNION ALL
SELECT device, 'Transactions', transactions FROM aggregated;
```

### 3. Scorecard Table (Wide Format)
> Used for dynamic scorecards on the second page – each metric is a separate column for use in metric tiles and filters.


```sql 
WITH base AS (
  SELECT
    CONCAT(fullVisitorId, CAST(visitId AS STRING)) AS session_id,
    device.deviceCategory AS device,
    totals.transactions AS transactions,
    hits.eCommerceAction.action_type AS action_type,
    hits.type AS hit_type,
    product.v2ProductName AS product_name
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
    UNNEST(hits) AS hits
    LEFT JOIN UNNEST(hits.product) AS product
  WHERE
    _TABLE_SUFFIX BETWEEN '20160801' AND '20160831'
)

SELECT
  device,
  COUNT(DISTINCT session_id) AS sessions,
  COUNT(DISTINCT IF(hit_type IN ('PAGE', 'EVENT') AND product_name IS NOT NULL, session_id, NULL)) AS product_views,
  COUNT(DISTINCT IF(CAST(action_type AS INT64) = 2, session_id, NULL)) AS add_to_cart,
  COUNT(DISTINCT IF(CAST(action_type AS INT64) = 3, session_id, NULL)) AS checkout,
  COUNT(DISTINCT IF(transactions >= 1, session_id, NULL)) AS transactions
FROM base
GROUP BY device
ORDER BY sessions DESC;
```


## Notes

- Product views are based on product.v2ProductName and relevant hit types (PAGE, EVENT)

- Conversion steps are counted per unique session

- Only includes data from August 2016

- Dataset is based on the anonymized Google Analytics 360 sample set
