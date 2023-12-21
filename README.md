# Retail-hypermarket-analysis

## Table of Content
1. [Dataset](https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/tree/main?tab=readme-ov-file#dataset)
2. [Problem Statement](https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/tree/main?tab=readme-ov-file#problem-statement)
3. [Analysis](https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/tree/main?tab=readme-ov-file#analysis)
4. [Recommendations](https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/tree/main?tab=readme-ov-file#recommendations)

## Dataset
The dataset consists of information about operations of a renowed Retail Company in Brazil and specifically 1,00,000 orders placed between 2016 & 2018. The dataset offers a comprehensive view of various dimensions including the order status, price, payment and freight performance, customer location, product attributes, and customer reviews. The data is spread across multiple CSVs which are as following:
1. customers.csv
2. sellers.csv
3. order_items.csv
4. geolocation.csv
5. payments.csv
6. reviews.csv
7. orders.csv
8. products.csv

The schema of the database looks as following:

<img src="https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/assets/127926279/a3dcdd01-27e3-47cf-bbbc-4e905d633070" width="800">

## Problem Statement
Task is to analyse data from various sources, extract valuable insights by looking at it combinedly & then provide actionable recommendations to improve the operations.

## Analysis
I used Google BigQuery data-warehouse to store & analyse data using [SQL queries](https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/blob/main/SQL-Queries.md). I also created a [Tableau Dashboard](https://public.tableau.com/app/profile/rahul2575/viz/Retailanalysis_16790631662820/Timedashboard) to visualize the important insights in form of charts.

Here are snippets of the some queries & their visualizations:
### What time do Brazilian customers tend to buy (Dawn, Morning, Afternoon or Night)?
### Dawn (2AM-5:59AM), Morning (6AM-11:59AM), Afternoon (12PM-5:59PM), Night (6PM-1:59AM)
```
WITH day_partitions AS (
  SELECT 
  CASE 
    WHEN EXTRACT(HOUR FROM order_purchase_timestamp) BETWEEN 2 AND 5 THEN 'Dawn'
    WHEN EXTRACT(HOUR FROM order_purchase_timestamp) BETWEEN 6 AND 11 THEN 'Morning'
    WHEN EXTRACT(HOUR FROM order_purchase_timestamp) BETWEEN 12 AND 17 THEN 'Afternoon'
    ELSE 'Night'
  END AS time_of_day
  FROM dataset.orders)

SELECT 
  time_of_day, 
  COUNT(*) AS orders_count
FROM day_partitions
GROUP BY time_of_day
ORDER BY orders_count DESC;
```
<img src="https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/assets/127926279/721898c8-e118-4574-99a0-e8c3b64d4b43" width="400">

### Top-10 combination of product categories that were purchased the most 
```
WITH sorted_data AS (
  SELECT 
    oi.order_id, 
    p.product_category
  FROM dataset.order_items AS oi LEFT JOIN dataset.products AS p ON oi.product_id = p.product_id
  WHERE p.product_category IS NOT NULL
  GROUP BY oi.order_id, p.product_category
), 
product_combinations AS (
  SELECT 
    order_id, 
    STRING_AGG(product_category, ", " ORDER BY product_category) AS product_concatenated
  FROM sorted_data
  GROUP BY order_id
  HAVING COUNT(DISTINCT product_category) > 1
)

SELECT
  product_concatenated, 
  COUNT(order_id) AS orders_count
FROM product_combinations
GROUP BY product_concatenated
ORDER BY orders_count DESC
LIMIT 10;
```
<img src="https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/assets/127926279/0ccec68c-45c1-45a2-85c1-1c47fdb7b7d2" width="800">

### Top-10 Best rated product categories
```
SELECT 
  p.product_category, 
  ROUND(AVG(r.review_score), 2) AS average_rating
FROM dataset.order_reviews AS r INNER JOIN dataset.order_items AS oi ON r.order_id = oi.order_id
     LEFT JOIN dataset.products AS p ON oi.product_id = p.product_id
GROUP BY p.product_category
ORDER BY average_rating DESC
LIMIT 10;
```
<img src="https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/assets/127926279/6d8b73ac-a09e-4a95-a7de-b4e0d18a321b" width="800">

### Mean & Sum of price and freight value by customer state
```
SELECT
  c.customer_state, 
  ROUND(SUM(oi.price), 2) AS total_sales, 
  ROUND(AVG(oi.price), 2) AS average_price, 
  ROUND(SUM(oi.freight_value), 2) AS total_freight, 
  ROUND(AVG(oi.freight_value), 2) AS average_freight
FROM dataset.order_items AS oi LEFT JOIN dataset.orders AS o ON oi.order_id = o.order_id
     INNER JOIN dataset.customers AS c ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY total_sales DESC, average_freight;
```
<img src="https://github.com/dataexplorerRahul/Retail-hypermarket-analysis/assets/127926279/fc3e91dc-9d94-4f74-b186-0464fef72235" width="600">


## Recommendations
* Focus on the order demand during the Afternoon hours like
  * Increase the availability and also visibility of products during those hours when
  the customer visits the site/app
  * Provide lucrative buying offers
  * Take care of the logistics beforehand so that the product is quickly dispatched
  as soon as it is ordered.
* Take steps so that the product reaches the customer either on or before the expected
delivery date. If the delivery happens way before the expected delivery date it leave a
positive impression on the customer (assuming the expected delivery date is
reasonable)
* Increasing the number of sellers in the states with higher contributions (i.e. SP, RJ, MG)
can also decrease the delivery time. RJ is even not in the Top 5 states with lowest
average delivery time.
* Provide more attractive offers for the Credit-card & UPI to boost their usage even more.
Also adding more options for payment-installments like No-cost EMI might boost the
number of repeat customers.
* Resolve the orders which have reviews related to late delivery, wrong product etc. and
make sure those mistakes aren’t repeated for the same customer again. Also, allowing
only verified customers to post reviews might help in getting genuine reviews of the
product & the service provided by Target.
* Provide more photos of the products on the site/app so that it becomes easier for the
customer to decide.
* Providing more offers for products in the following categories might lead to boosting
the overall sales:
Bed-table-bath, Health Beauty, Sport Leisure, Furniture Decoration, Computer
Accessories.
