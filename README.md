# Overview
>
This project use data of Wide World Importers company to have an overview of sales performance and analyze detailed customer segments to propose appropriate actions.
>
# Dataset
> This dataset is about a fictitious company Wide World Importers, provided by Microsoft. Wide World Importers (WWI) is a wholesale novelty goods importer and distributor operating from the San Francisco bay area.
The dataset contains information about customers, products, stock and sales of WWI between 2013-01-01 and 2016-05-31. (source : Wide World Importers [dataset](https://dataedo.com/samples/html2/WideWorldImporters/#/doc/d5/wideworldimporters))

# Tools
>
- SQL Bigquery
- Looker Studio
# Cohort Analysis & Customer Segmentation with RFM

**Cohort Analysis** is method to group customers based on shared characteristics within a specific timeframe. In cohort analysis, customers are divided into distinct groups or cohorts based on factors such as their acquisition date, behavior, or demographics. By studying these cohorts, businesses can gain insights into customer retention, engagement, and behavior patterns over time.
>In this analysis, customers will be segmented based on the date of their first product purchase, and this segmentation will be performed on a monthly basis to determine how many of them (%) coming back for the following month in 2013-2016.
>
>
This Analysis is performed based on analytic data models, which is a part of Data Warehouse project.
The project includes: clean raw data, build dim and fact tables, analyze customer and sales.

# Exploratory Data Analysis
**1. Cohort Analysis**
- Sales period: create data of year and month for each single transaction for each customer
```sql
WITH fact_sales__source AS (
  SELECT customer_key
    , order_date
  FROM `data-warehouse-course-391316`.`wide_world_importers_dwh`.`fact_sales_order_line`
)

, fact_sales__transaction AS (
  SELECT DISTINCT
    DATE_TRUNC(order_date, MONTH) AS order_month
    , customer_key
  FROM fact_sales__source
)

, fact_sales__first_latest_order_month AS (
  SELECT 
    customer_key
    , DATE_TRUNC(MIN(order_date), MONTH) AS first_order_month
    , DATE_TRUNC(MAX(order_date), MONTH) AS latest_order_month
  FROM fact_sales__source
  GROUP BY 1
)
```
- Cohort group: year and month of a customer’s first purchase

```sql

-- Count number of customers that have same first purchase month as Cohort size
, fact_cohort__cohort_size  as (
  SELECT 
    first_order_month AS cohort_month
    , count(customer_key) AS cohort_size
  FROM fact_sales__first_latest_order_month
  GROUP BY 1
)
, fact_cohort__retention AS (
  SELECT  
    fact_sales__first_latest_order_month.first_order_month AS cohort_month
    , fact_sales.order_month
    , customer_key
  FROM fact_sales__transaction  AS fact_sales
  LEFT JOIN fact_sales__first_latest_order_month USING (customer_key)
)
```

> *SITUATION*: we are creating a model to calculate number of active user in each month for each cohort
>
> *PROBLEM*: if there is no transaction in any order month of a specific cohort => it will be missing lines for those order month, it must be display this month with count = 0
>
> *SOLUTION*: we will handle sparse data by making it densed. By Cross join vs dim_year_month we will create every order month for each cohort, then use Left join to handle missing transactions
```sql

, dim_year_month AS (
  SELECT DISTINCT year_month
  FROM `data-warehouse-course-391316`.`wide_world_importers_dwh`.`dim_date`
)

, fact_cohort__densed AS (
  SELECT DISTINCT 
    first_order_month AS cohort_month
    , year_month AS order_month
  FROM fact_sales__first_latest_order_month 
  CROSS JOIN dim_year_month
  WHERE year_month BETWEEN first_order_month AND latest_order_month
)
-- Count monthly active customers from each cohort
, fact_cohort__retention_densed AS (
  SELECT  
    cohort_month
    , order_month
    , DATE_DIFF(order_month, cohort_month, MONTH)  AS period
    , COUNT(customer_key) AS active_user
  FROM fact_cohort__densed
  LEFT JOIN fact_cohort__retention USING (cohort_month, order_month)
  GROUP BY 1,2,3
)

SELECT
  cohort_month
  , fact_cohort__retention.period
  , fact_cohort__cohort_size.cohort_size
  , fact_cohort__retention.active_user
  , fact_cohort__retention.active_user*100 / fact_cohort__cohort_size.cohort_size as percentage --to show the number as percentage 
FROM fact_cohort__retention_densed AS fact_cohort__retention
LEFT JOIN fact_cohort__cohort_size USING (cohort_month)
ORDER BY 1,2
```
**Table Result**
>
![image](https://github.com/thupham16/Analysis/assets/119646834/bb085328-d486-4af7-a11e-e604b3216c3d)
>
This query result was then connected to Looker Studio to converted into pivot table for analysis. Link to Looker Studio here
>
>![image](https://github.com/thupham16/Analysis/assets/119646834/ea1c5c86-5926-46d5-b6a2-942cf4ad18eb)

**Insight**
>
**Recommendations**

**2. RFM Analysis**

**Recency, Frequency and Monetary (RFM)** analysis is a technique to gain insights into customer spending patterns and effectively segment them, enable business to determine their target audience and devise tailored marketing campaigns.
- Recency: how recently a customer purchased a product, by subtracting the date of the latest order date from the date of the analysis. The more recently a customer has shopped, the more likely they are to keep your company in mind for future purchases.
- Frequency: total number of purchases - indicate how often a customer purchase to predict when they will comback or to remind customers of their needs.
- Monetary value: to calculate how much money they spend, that help company to identify customer who spend the most versus relatively small amounts.
>
Group the data by customer and calculate Recency, Frequency and Monetary metrics:
```sql
WITH fact_rfm__summary AS (
  SELECT 
    customer_key
    , MAX(order_date) AS last_active_date
    , DATE_DIFF('2016-06-1', MAX(order_date), MONTH) AS recency --Since recency is calculated for a point in time and the dataset last order date is May 31 2016, we will set the 1 day after to calculate recency. 
    , COUNT (DISTINCT sales_order_key) AS frequency
    , SUM(gross_amount) AS monetary
  FROM `data-warehouse-course-391316`.`wide_world_importers_dwh`.`fact_sales_order_line`
  GROUP BY 1
)
```
RFM values will be scored and segmented by Percentiles. The score ranges from 1 to 5, where the higher this number, the better:
- The more recent the customer's purchase the higher the Recency (R) score.
- The more purchases the customer makes, the higher the Frequency score (F)
- The more the customer spends on purchases, the higher the score the customer will have Monetarity(M)
```sql
, fact_rfm__percentile AS (
  SELECT
    *
    , PERCENT_RANK() OVER (ORDER BY recency DESC) AS recency_rank --the closer active date, the higher rank for recency
    , PERCENT_RANK() OVER (ORDER BY frequency) AS frequency_rank
    , PERCENT_RANK() OVER (ORDER BY monetary) AS monetary_rank
  FROM fact_rfm__summary
)

, fact_frm__score AS (
  SELECT  *
    , CASE 
      WHEN recency_rank BETWEEN 0 AND 0.2 THEN 1
      WHEN recency_rank BETWEEN 0.2 AND 0.4 THEN 2
      WHEN recency_rank BETWEEN 0.4 AND 0.6 THEN 3
      WHEN recency_rank BETWEEN 0.6 AND 0.8 THEN 4
      WHEN recency_rank > 0.8 THEN 5
      ELSE 0
     END AS recency_score
    , CASE 
      WHEN frequency_rank BETWEEN 0 AND 0.2 THEN 1
      WHEN frequency_rank BETWEEN 0.2 AND 0.4 THEN 2
      WHEN frequency_rank BETWEEN 0.4 AND 0.6 THEN 3
      WHEN frequency_rank BETWEEN 0.6 AND 0.8 THEN 4
      WHEN frequency_rank > 0.8 THEN 5
      ELSE 0
     END AS frequency_score
    , CASE 
      WHEN monetary_rank BETWEEN 0 AND 0.2 THEN 1
      WHEN monetary_rank BETWEEN 0.2 AND 0.4 THEN 2
      WHEN monetary_rank BETWEEN 0.4 AND 0.6 THEN 3
      WHEN monetary_rank BETWEEN 0.6 AND 0.8 THEN 4
      WHEN monetary_rank > 0.8 THEN 5
      ELSE 0
      END AS monetary_score
  FROM fact_rfm__percentile
)
```
>
- Customers who are frequent buyers, have recently shopped and usually are spending a lot of money, they would get a score of 555: Recency (R) – 5, Frequency (F) – 5, Monetary (M) – 5. They are your best customers.
>
- On the other hand, customers who spend the least, do almost no buying at all and their last purchase was really long time ago, they will get a score of 111: Recency (R) – 1, Frequency (F) – 1, Monetary (M) – 1.
Grouping Customers into RFM segment and category
```sql
, fact_rfm__segment AS (
  SELECT *
    , CAST(CONCAT(recency_score, frequency_score, monetary_score) AS INT64) AS rfm_segment
  FROM fact_frm__score
)

  SELECT *
    , CASE
      WHEN rfm_segment IN (555, 554, 544, 545, 454, 455, 445) THEN "Champions"
      WHEN rfm_segment IN (543, 444, 435, 355, 354, 345, 344, 335) THEN "Loyal Customers"
      WHEN rfm_segment IN (553, 551, 552, 541, 542, 533, 532, 531, 452, 451, 442, 441, 431, 453, 433, 432, 423, 353, 352, 351, 342, 341, 333, 323) THEN "Potential Loyalist"
      WHEN rfm_segment IN (512, 511, 422, 421, 412, 411, 311) THEN "Recent Customers"
      WHEN rfm_segment IN (525, 524, 523, 522, 521, 515, 514, 513, 425, 424, 413,414, 415, 315, 314, 313) THEN "Promising"
      WHEN rfm_segment IN (535, 534, 443, 434, 343, 334, 325, 324) THEN "Customers Needing Attention"
      WHEN rfm_segment IN (331, 321, 312, 221, 213) THEN "About To Sleep"
      WHEN rfm_segment IN (255, 254, 245, 244, 253, 252, 243, 242, 235, 234, 225, 224, 153, 152, 145, 143, 142, 135, 134, 133, 125, 124) THEN "At Risk"
      WHEN rfm_segment IN (155, 154, 144, 214, 215,115, 114, 113) THEN "Can’t Lose Them"
      WHEN rfm_segment IN (332, 322, 231, 241, 251, 233, 232, 223, 222, 132, 123, 122, 212, 211) THEN "Hibernating"
      WHEN rfm_segment IN (111, 112, 121, 131, 141,151) THEN "Lost"
    ELSE 'Undefined'
    END AS rfm_category
  FROM fact_rfm__segment
```
**Table Result**
> ![image](https://github.com/thupham16/Analysis/assets/119646834/2d725e91-d2e1-4109-abbe-b17c7abb1845)

> ![image](https://github.com/thupham16/Analysis/assets/119646834/d2d334fd-dede-4ad1-8be9-d3861702218b)


