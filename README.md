# 📊 Ecommerce Traffic & Conversion Performance Analysis | SQL, BigQuery

**Author:** Hoang Duc Kien

**Date:** 2025

**Tools Used:** SQL, BigQuery

## 📑 Table of Contents

- [📌 Background & Overview](#-background--overview)
- [📂 Dataset Description & Data Structure](#-dataset-description--data-structure)
- [⚒️ Main Analysis Process](#️-main-analysis-process)
- [🔎 Final Insights & Recommendations](#-final-insights--recommendations)

## 📌 Background & Overview

### 📖 What is this project about?

This project analyzes user traffic and purchasing behavior of an e-commerce website using SQL in BigQuery.

The objective is to evaluate:

- ✔️ Traffic growth trends
- ✔️ Conversion efficiency
- ✔️ User purchasing patterns
- ✔️ Marketing channel effectiveness

The analysis focuses on understanding how users move through the purchase funnel and identifying optimization opportunities to increase revenue.

### 👤 Target Audience

- ✔️ Data Analysts & Business Analysts
- ✔️ Digital Marketing Teams
- ✔️ E-commerce Managers & Stakeholders
- ✔️ Business Intelligence Teams

## 📂 Dataset Description & Data Structure

**📌 Data Source:**  
Public Google Analytics dataset stored in BigQuery.

**📌 Dataset Used:**  
`bigquery-public-data.google_analytics_sample`

**📌 Key Tables:**

- `ga_sessions_2017*`

**📌 Key Fields Used:**


- `fullVisitorId`
- `date`
- `totals`
- `totals.bounces`
- `totals.hits`
- `totals.pageviews`
- `totals.visits`
- `totals.transactions`
- `trafficSource.source`
- `hits`
- `hits.eCommerceAction`
- `hits.eCommerceAction.action_type`
- `hits.product`
- `hits.product.productQuantity`
- `hits.product.productRevenue`
- `hits.product.productSKU`
- `hits.product.v2ProductName`

## ⚒️ Main Analysis Process

### 🔍 Query 01: Calculate total visit, pageview, transaction for Jan, Feb and March 2017 (order by month)

#### 🚀 Query

```sql
SELECT 
  FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS month,
  SUM(totals.visits) AS visits,
  SUM(totals.pageviews) AS pageviews,
  SUM(totals.transactions) AS transactions 
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix BETWEEN '0101' AND '0331' 
GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))
ORDER BY month;
```

#### 💡 Result

### 🔍 Query 02: Bounce rate per traffic source in July 2017

Bounce_rate = num_bounce/total_visit (order by total_visit DESC)

#### 🚀 Query

```sql
With raw_data AS(
  SELECT 
    trafficSource.source as source,
    SUM(totals.visits) as total_visits,
    SUM(totals.bounces) as total_no_of_bounces
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
  GROUP BY trafficSource.source
  ORDER BY total_visits DESC
)

SELECT 
  source,
  total_visits,
  total_no_of_bounces,
  ROUND((total_no_of_bounces/total_visits)*100,3) AS bounce_rate
FROM raw_data;
```

#### 💡 Result

### 🔍 Query 03: Revenue by traffic source by week, by month in June 2017

#### 🚀 Query

```sql
WITH revenue_week AS(
  SELECT
    'Week' as time_type,
    FORMAT_DATE('%G%V', PARSE_DATE('%Y%m%d', date)) AS time,
    trafficSource.source,
    SUM(productRevenue/1000000) as revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE productRevenue IS NOT NULL
  GROUP BY FORMAT_DATE('%G%V', PARSE_DATE('%Y%m%d', date)),trafficSource.source
),
revenue_month AS(
    SELECT
    'Month' as time_type,
    FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS time,
    trafficSource.source,
    SUM(productRevenue/1000000) as revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE productRevenue IS NOT NULL
  GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)),trafficSource.source
)

SELECT
  time_type,
  time,
  source,
  ROUND(revenue,4)
FROM(
  SELECT * FROM revenue_week
  UNION ALL
  SELECT * FROM revenue_month
)
ORDER BY revenue DESC;
```

#### 💡 Result

### 🔍 Query 04: Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017

#### 🚀 Query

```sql
WITH purchase_data AS(
  SELECT 
    FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS month,
    SUM(totals.pageviews) AS total_pageviews,
    COUNT (DISTINCT fullVisitorId) AS number_unique_user
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE 
    _table_suffix between '0601' and '0731'AND
    totals.transactions >= 1 AND
    productRevenue IS NOT NULL
  GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))
),
non_purchase_data AS(
  SELECT 
    FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS month,
    SUM(totals.pageviews) AS total_pageviews,
    COUNT (DISTINCT fullVisitorId) AS number_unique_user
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE 
    _table_suffix between '0601' and '0731'and
    totals.transactions IS NULL AND
    productRevenue IS NULL
  GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))
)

SELECT
  p.month,
  ROUND(p.total_pageviews/p.number_unique_user,8) AS avg_pageviews_purchase,
  ROUND(n.total_pageviews/n.number_unique_user,7) AS avg_pageviews_non_purchase
FROM purchase_data AS p
INNER JOIN non_purchase_data AS n
ON p.month = n.month
ORDER BY month;
```

#### 💡 Result

### 🔍 Query 05: Average number of transactions per user that made a purchase in July 2017

#### 🚀 Query

```sql
WITH purchase_tran_data AS(
  SELECT 
    FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS Month,
    SUM(totals.transactions) AS total_transactions,
    COUNT (DISTINCT fullVisitorId) AS number_unique_user
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE
    totals.transactions >= 1 AND
    productRevenue IS NOT NULL
  GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))
)

SELECT
  Month,
  ROUND(total_transactions/number_unique_user,9) AS Avg_total_transactions_per_user
FROM purchase_tran_data;
```

#### 💡 Result

### 🔍 Query 06: Average amount of money spent per session. Only include purchaser data in July 2017

#### 🚀 Query

```sql
WITH amount_money_per_session AS(
  SELECT 
    FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) AS Month,
    SUM(totals.visits) AS total_visits,
    SUM(productRevenue)/1000000 AS total_revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE
    totals.transactions >= 1 AND
    productRevenue IS NOT NULL
  GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))
)

SELECT
  Month,
  ROUND(total_revenue/total_visits,2) AS avg_revenue_by_user_per_visit
FROM amount_money_per_session;
```

#### 💡 Result

### 🔍 Query 07: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017

Output should show product name and the quantity was ordered.

#### 🚀 Query

```sql
WITH list_customer AS(
  SELECT 
    fullVisitorId,
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE 
    eCommerceAction.action_type = '6'AND
    v2ProductName = "YouTube Men's Vintage Henley"
)
SELECT 
  v2ProductName AS other_purchased_products,
  SUM(productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST(hits)hits,
UNNEST(hits.product)
WHERE
  totals.transactions >= 1 AND
  productRevenue IS NOT NULL AND
  fullVisitorId IN (SELECT fullVisitorId FROM list_customer) AND
  v2ProductName <> "YouTube Men's Vintage Henley"
GROUP BY v2ProductName
ORDER BY quantity DESC;
```

#### 💡 Result

### 🔍 Query 08: Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017

For example, 100% product view then 40% add_to_cart and 10% purchase.  
Add_to_cart_rate = number product add to cart/number product view.  
Purchase_rate = number product purchase/number product view.  
The output should be calculated in product level.

#### 🚀 Query

```sql
WITH raw_data AS(
  SELECT 
    FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date)) as month,
    SUM(CASE WHEN eCommerceAction.action_type = "2" THEN 1 ELSE 0 END) AS num_product_view,
    SUM(CASE WHEN eCommerceAction.action_type = "3" THEN 1 ELSE 0 END) AS num_addtocart,
    SUM(CASE WHEN eCommerceAction.action_type = "6" AND  productRevenue IS NOT NULL THEN 1 ELSE 0 END) AS num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits)hits,
  UNNEST(hits.product)
  WHERE 
    _table_suffix between '0101' and '0331'
  GROUP BY FORMAT_DATE('%Y%m',PARSE_DATE('%Y%m%d', date))
)
SELECT
  month,
  num_product_view,
  num_addtocart,
  num_purchase,
  ROUND((num_addtocart/num_product_view)*100,2) AS add_to_cart_rate,
  ROUND((num_purchase/num_product_view)*100,2) AS purchase_rate
FROM raw_data
ORDER BY month;
```

#### 💡 Result

## 🔎 Final Insights & Recommendations

### 📌 Key Insights

- Conversion rate remains relatively stable across Q1 but varies significantly by traffic source.
- Revenue concentration is heavily dependent on a few major channels.
- Purchasers exhibit higher engagement (pageviews & time on site).
- Repeat buyers represent a smaller segment but contribute disproportionately to revenue.
- Funnel analysis shows major drop-offs between product view and add-to-cart stage.

### 📌 Strategic Recommendations

**Improve Add-to-Cart Experience**  
Optimize product pages and call-to-action placement to reduce funnel drop-off.

**Invest in High-Converting Channels**  
Allocate marketing budget toward channels with strong conversion performance.

**Retarget Engaged Non-Purchasers**  
Focus remarketing campaigns on high pageview users without transactions.

**Develop Loyalty Strategy**  
Encourage repeat purchases through promotions, membership programs, and personalized offers.