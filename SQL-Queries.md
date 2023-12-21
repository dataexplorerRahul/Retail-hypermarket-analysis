## Datatype of columns in customers table
```
SELECT column_name, data_type
FROM dataset.INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'customers';
```

## Time period for which the data is given
```
SELECT 
  FORMAT_DATE("%b-%e-%Y", MIN(order_purchase_timestamp)) AS first_order, 
  FORMAT_DATE("%b-%e-%Y", MAX(order_purchase_timestamp)) AS last_order
FROM dataset.orders;
```

## Is there a growing trend on e-commerce in Brazil?
```
SELECT 
  EXTRACT(YEAR FROM order_purchase_timestamp) AS year, 
  EXTRACT(MONTH FROM order_purchase_timestamp) AS month, 
  COUNT(order_id) AS orders_count
FROM dataset.orders
GROUP BY year, month
ORDER BY year, month;
```

## What time do Brazilian customers tend to buy (Dawn, Morning, Afternoon or Night)?
## Dawn (2AM-5:59AM), Morning (6AM-11:59AM), Afternoon (12PM-5:59PM), Night (6PM-1:59AM)
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

## Get month on month orders by states
```
SELECT 
  c.customer_state,
  EXTRACT(YEAR FROM o.order_purchase_timestamp) AS year, 
  EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month, 
  COUNT(o.order_id) AS orders_count
FROM dataset.orders AS o LEFT JOIN dataset.customers AS c ON o.customer_id = c.customer_id
GROUP BY year, month, c.customer_state
ORDER BY year, month, orders_count DESC;
```

## Distribution of customers across the states in Brazil
```
SELECT 
  c.customer_state, 
  COUNT(c.customer_id) AS total_customers
FROM dataset.orders AS o INNER JOIN dataset.customers AS c ON o.customer_id = c.customer_id
GROUP BY customer_state
ORDER BY total_customers DESC;
```

## Get % increase in cost of orders from 2017 to 2018 (include months between Jan to Aug only) 
```
WITH summary_2017 AS (
  SELECT
    EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month, 
    ROUND(SUM(p.payment_value), 2) AS total_payments
  FROM dataset.payments AS p LEFT JOIN dataset.orders AS o ON p.order_id = o.order_id
  WHERE (EXTRACT(YEAR FROM o.order_purchase_timestamp) = 2017) AND (EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8)
  GROUP BY month
  ORDER BY month), 
summary_2018 AS (
  SELECT
    EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month, 
    ROUND(SUM(p.payment_value), 2) AS total_payments
  FROM dataset.payments AS p LEFT JOIN dataset.orders AS o ON p.order_id = o.order_id
  WHERE (EXTRACT(YEAR FROM o.order_purchase_timestamp) = 2018) AND (EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8)
  GROUP BY month
  ORDER BY month
)

SELECT
  s1.month, 
  s1.total_payments AS sales_2017, 
  s2.total_payments AS sales_2018, 
  ROUND((s2.total_payments - s1.total_payments)*100/s1.total_payments, 2) AS percentage_inc
FROM summary_2017 AS s1 INNER JOIN summary_2018 AS s2 ON s1.month = s2.month
ORDER BY s1.month;
```

## Mean & Sum of price and freight value by customer state
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

## Analysis on freight and delivery time
```
CREATE VIEW dataset.delivery_summary AS
SELECT
	c.customer_state, 
	ROUND(AVG(DATE_DIFF(DATE(o.order_delivered_customer_date), DATE(o.order_purchase_timestamp), DAY)), 2) AS time_to_delivery,
	ROUND(AVG(DATE_DIFF(DATE(o.order_estimated_delivery_date), DATE(o.order_purchase_timestamp), DAY)), 2) AS diff_estimated_delivery,
	ROUND(AVG(oi.freight_value), 2) AS average_freight
FROM dataset.order_items AS oi LEFT JOIN dataset.orders AS o ON oi.order_id = o.order_id
	INNER JOIN dataset.customers AS c ON o.customer_id = c.customer_id
GROUP BY c.customer_state;
```

## Top-5 states with highest average freight
```
SELECT *
FROM dataset.delivery_summary
ORDER BY average_freight DESC
LIMIT 5;
```

## Top-5 states with lowest average freight
```
SELECT *
FROM dataset.delivery_summary
ORDER BY average_freight 
LIMIT 5;
```

## Top-5 states with highest time_to_delivery
```
SELECT *
FROM dataset.delivery_summary
ORDER BY time_to_delivery DESC
LIMIT 5;
```

## Top-5 states with lowest time_to_delivery
```
SELECT *
FROM dataset.delivery_summary
ORDER BY time_to_delivery
LIMIT 5;
```

## Top-5 states with fastest delivery compared to expected date
```
SELECT *
FROM dataset.delivery_summary
ORDER BY diff_estimated_delivery - time_to_delivery DESC
LIMIT 5;
```

## Top-5 states with slowest delivery compared to expected date
```
SELECT *
FROM dataset.delivery_summary
ORDER BY diff_estimated_delivery - time_to_delivery 
LIMIT 5;
```
## Month over Month count of orders for different payment types
```
SELECT 
  EXTRACT(YEAR FROM o.order_purchase_timestamp) AS year, 
  EXTRACT(MONTH FROM o.order_purchase_timestamp) AS month, 
  SUM(IF(p.payment_type = 'credit_card', 1, 0)) AS credit_card_orders, 
  SUM(IF(p.payment_type = 'voucher', 1, 0)) AS voucher_orders, 
  SUM(IF(p.payment_type = 'debit_card', 1, 0)) AS debit_card_orders, 
  SUM(IF(p.payment_type = 'UPI', 1, 0)) AS UPI_orders, 
  SUM(IF(p.payment_type = 'not_defined', 1, 0)) AS not_defined_orders
FROM dataset.payments AS p LEFT JOIN dataset.orders AS o ON p.order_id = o.order_id
GROUP BY year, month
ORDER BY year, month;
```

## Count of orders based on the no. of payment installments
```
SELECT
  payment_installments,
  COUNT(order_id) AS orders_count
FROM dataset.payments
GROUP BY payment_installments
ORDER BY orders_count DESC;

SELECT 
  num_installments, 
  COUNT(order_id) AS orders_count
FROM 
(SELECT
  order_id,
  MAX(payment_installments) AS num_installments
FROM dataset.payments
GROUP BY order_id) AS x
GROUP BY num_installments
ORDER BY num_installments;
```

## Top-10 cities from where customers placed the orders
```
SELECT
  c.customer_city, 
  COUNT(order_id) AS orders_count
FROM dataset.orders AS o LEFT JOIN dataset.customers AS c ON o.customer_id = c.customer_id
GROUP BY c.customer_city
ORDER BY orders_count DESC
LIMIT 10;
```

## States with faster deliveries than promised? Average difference between delivery & expected dates
```
SELECT
  customer_state, 
  diff_estimated_delivery, 
  time_to_delivery,
  ROUND(diff_estimated_delivery - time_to_delivery, 2) AS faster_delivery
FROM dataset.delivery_summary
ORDER BY faster_delivery DESC
LIMIT 10;
```

## Average time to approve an order
```
SELECT
  ROUND(AVG(TIMESTAMP_DIFF(order_approved_at, order_purchase_timestamp, SECOND))/(60*60), 2) AS avg_approval_time_hrs
FROM dataset.orders;
```

## What is the cancellation rate of orders & unavailability of orders
```
SELECT
  ROUND(SUM(IF(order_status = 'canceled', 1, 0))*100/COUNT(*), 2) AS cancellation_rate,
  ROUND(SUM(IF(order_status = 'unavailable', 1, 0))*100/COUNT(*), 2) AS unavailability_rate 
FROM dataset.orders;
```

## Count number of orders based on number days of taken to deliver
```
SELECT
  DATE_DIFF(order_delivered_customer_date, order_purchase_timestamp, DAY) AS days_to_deliver,
  COUNT(order_id) AS orders_count
FROM dataset.orders
WHERE order_delivered_customer_date IS NOT NULL
GROUP BY days_to_deliver
ORDER BY orders_count DESC;
```

## Check the number of sellers in each order
```
SELECT
  order_id, 
  COUNT(DISTINCT seller_id) AS num_sellers
FROM dataset.order_items
GROUP BY order_id
ORDER BY num_sellers DESC;
```

## Number of items in each order
```
SELECT 
  total_items, 
  COUNT(order_id) AS orders_count
FROM 
(SELECT
  order_id, 
  COUNT(*) AS total_items
FROM dataset.order_items
GROUP BY order_id) AS x
GROUP BY total_items
ORDER BY orders_count DESC;
```

## Number of different products in each order
```
SELECT 
  num_products, 
  COUNT(order_id) AS orders_count
FROM 
(SELECT
  order_id, 
  COUNT(DISTINCT product_id) AS num_products
FROM dataset.order_items
GROUP BY order_id) AS x
GROUP BY num_products
ORDER BY orders_count DESC;
```

## Where do the Top-10 sellers live for the city that orders most
```
WITH base AS (
  SELECT *
  FROM dataset.orders AS o LEFT JOIN dataset.customers AS c ON o.customer_id = c.customer_id
      LEFT JOIN dataset.order_items AS oi ON o.order_id = oi.order_id
      INNER JOIN dataset.sellers AS s ON oi.seller_id = s.seller_id), 
most_orders_city AS (
  SELECT 
    customer_city,
    COUNT(*) AS orders_count
  FROM base
  GROUP BY customer_city
  ORDER BY orders_count DESC
  LIMIT 1), 
top_seller_cities AS (
  SELECT 
    seller_city, 
    COUNT(customer_city) AS num_orders, 
    customer_city
  FROM base
  WHERE customer_city IN (SELECT customer_city FROM most_orders_city)
  GROUP BY seller_city, customer_city 
  ORDER BY num_orders DESC
  LIMIT 5)

SELECT
  customer_city, 
  STRING_AGG(seller_city, ', ') AS top_5_seller_cities
FROM top_seller_cities
GROUP BY customer_city;
```

## Sum of payments made by customer in each state and what % of it are price & freight values?
```
SELECT
  c.customer_state, 
  ROUND(SUM(oi.price + oi.freight_value), 2) AS total_sales, 
  ROUND(SUM(oi.price)*100/SUM(oi.price + oi.freight_value), 2) AS price_perc_of_sales,
  ROUND(SUM(oi.freight_value)*100/SUM(oi.price + oi.freight_value), 2) AS freight_perc_of_sales, 
FROM dataset.order_items AS oi LEFT JOIN dataset.orders AS o ON oi.order_id = o.order_id
     INNER JOIN dataset.customers AS c ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY total_sales DESC, freight_perc_of_sales DESC;
```

## Top-10 product category by Purchase
```
SELECT 
  p.product_category, 
  COUNT(oi.order_id) AS purchase_count, 
  ROUND(AVG(r.review_score), 2) AS average_rating
FROM dataset.order_reviews AS r INNER JOIN dataset.order_items AS oi ON r.order_id = oi.order_id
     LEFT JOIN dataset.products AS p ON oi.product_id = p.product_id
GROUP BY p.product_category
ORDER BY purchase_count DESC
LIMIT 10;
```

## Top-10 Best rated product categories
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

## Top-10 Worst rated product categories
```
SELECT 
  p.product_category, 
  ROUND(AVG(r.review_score), 2) AS average_rating
FROM dataset.order_reviews AS r INNER JOIN dataset.order_items AS oi ON r.order_id = oi.order_id
     LEFT JOIN dataset.products AS p ON oi.product_id = p.product_id
GROUP BY p.product_category
ORDER BY average_rating 
LIMIT 10;
```

## Top-10 combination of product categories that were purchased the most
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
