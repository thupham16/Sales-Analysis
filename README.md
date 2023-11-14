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

# Output
- Cleansed Data
- Data Modelling
- Visualization
- Insight and Recommendations
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



