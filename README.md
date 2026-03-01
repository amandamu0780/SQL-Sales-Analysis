# Objective
This project aims to demonstrate my intermediate SQL skills by identifying key customers and their buying behavior for an e-commerce company utilizing RFM analysis. It provides insights into geographic data, the most profitable times of the year, and product performance. The purpose of this work is to inform potential areas of focus on specific customers, products, and locations. The following provides detailed objectives, business problems, and its solutions and findings.
## Dataset
The data of this project was sourced from Kaggle: Ecommerce_Data
## Creating Tables
```sql
CREATE TABLE order_details (
    Order_id VARCHAR(25),
    Amount DECIMAL(10,2),
    Profit DECIMAL(10,2),
    Quantity INT,
    Category VARCHAR(25),
    Sub_category VARCHAR(25)
);
CREATE TABLE order_details (
		Order_id VARCHAR(25),
		Amount DECIMAL (10,2),
		Profit DECIMAL(10,2),
		Quantity INT,
		Category VARCHAR (25),
		Sub_category VARCHAR (25)
);

CREATE TABLE sales_target (
		Month_order VARCHAR(25),
		Category VARCHAR (25),
		Target DECIMAL (10,2)
);
```
## Business Problems and Solutions
1) What are the most profitable Seasons in India?
```sql
WITH months AS 
(
SELECT 
EXTRACT (month FROM lo.order_date) AS month,
od.profit
FROM order_details od
JOIN list_orders lo
ON od.order_id = lo.order_id
GROUP BY 1,2
)
SELECT 
SUM(profit) AS total_profit,
CASE 
WHEN month IN (2,3) THEN 'Spring'
WHEN month IN (4,5) THEN 'Summer'
WHEN month IN (6,7,8,9) THEN 'Monsoon'
WHEN month IN (10,11) THEN 'Fall'
WHEN month IN (12,1) THEN 'Winter'
END AS Season
FROM months
GROUP BY 2
ORDER BY 1 DESC;
```
### Output
| Season   | Total Profit |
|----------|--------------|
| Spring   | 13403.00     |
| Fall     | 12601.00     |
| Winter   | 12029.00     |
| Summer   | -6957.00     |
| Monsoon  | -13249.00    |

2) Find 10 most profitable products
```sql
SELECT 
od.category,
od.sub_category,
SUM(od.profit)
FROM order_details od
JOIN list_orders lo
ON od.order_id = lo.order_id
GROUP BY 1,2
ORDER BY 3 DESC
LIMIT 10;
```
## Profit by Category and Subcategory

| Category     | Subcategory  | Total Sales |
|--------------|--------------|-------------|
| Electronics  | Printers     | 5964.00     |
| Furniture    | Bookcases    | 4888.00     |
| Electronics  | Accessories  | 3559.00     |
| Clothing     | Trousers     | 2847.00     |
| Clothing     | Stole        | 2559.00     |
| Electronics  | Phones       | 2207.00     |

3) Most Profitable States
```sql
SELECT 
lo.state,
SUM(od.profit)
FROM order_details od
JOIN list_orders lo
ON od.order_id = lo.order_id
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```
## Sales by State

| State            | Total Sales |
|------------------|------------|
| Maharashtra      | 6176.00    |
| Madhya Pradesh   | 5551.00    |
| Uttar Pradesh    | 3237.00    |
| Delhi            | 2987.00    |
| West Bengal      | 2500.00    |
| Kerala           | 1871.00    |

## Customer Segmentation
### How can we Segment our Customers based on RFM scores
#### Calculating RFM scores
```sql
SELECT
        lo.customer_name,
        od.order_id AS order_id,
        AGE(CURRENT_DATE, MAX(lo.order_date)) AS Recency,
        COUNT(od.order_id) AS Frequency,
        SUM(od.amount) AS Monetary,
        NTILE(5) OVER (ORDER BY AGE(CURRENT_DATE, MAX(lo.order_date)) DESC) AS R,
        NTILE(5) OVER (ORDER BY COUNT(od.order_id) ASC) AS F,
        NTILE(5) OVER (ORDER BY SUM(od.amount) ASC) AS M
    FROM 
        order_details od
    INNER JOIN 
        list_orders lo ON od.order_id = lo.order_id
    GROUP BY 
        lo.customer_name, od.order_id
```
#### Combining RFM scores
```sql
WITH Rfm AS (
    SELECT
        lo.customer_name,
        od.order_id AS order_id,
        AGE(CURRENT_DATE, MAX(lo.order_date)) AS Recency,
        COUNT(od.order_id) AS Frequency,
        SUM(od.amount) AS Monetary,
        NTILE(5) OVER (ORDER BY AGE(CURRENT_DATE, MAX(lo.order_date)) DESC) AS R,
        NTILE(5) OVER (ORDER BY COUNT(od.order_id) ASC) AS F,
        NTILE(5) OVER (ORDER BY SUM(od.amount) ASC) AS M
    FROM 
        order_details od
    INNER JOIN 
        list_orders lo ON od.order_id = lo.order_id
    GROUP BY 
        lo.customer_name, od.order_id
)
SELECT 
		order_id,
        customer_name,
        CONCAT(R, F, M) AS Rfm_score
    FROM 
        Rfm 
    ORDER BY 
        Rfm_score DESC
```
#### Categorizing Customers based in their RFM scores
```sql
WITH Rfm AS (
    SELECT
        lo.customer_name,
        od.order_id AS order_id,
        AGE(CURRENT_DATE, MAX(lo.order_date)) AS Recency,
        COUNT(od.order_id) AS Frequency,
        SUM(od.amount) AS Monetary,
        NTILE(5) OVER (ORDER BY AGE(CURRENT_DATE, MAX(lo.order_date)) DESC) AS R,
        NTILE(5) OVER (ORDER BY COUNT(od.order_id) ASC) AS F,
        NTILE(5) OVER (ORDER BY SUM(od.amount) ASC) AS M
    FROM 
        order_details od
    INNER JOIN 
        list_orders lo ON od.order_id = lo.order_id
    GROUP BY 
        lo.customer_name, od.order_id
),
Rfm_scores AS (
    SELECT 
		order_id,
        customer_name,
        CONCAT(R, F, M) AS Rfm_score
    FROM 
        Rfm 
    ORDER BY 
        Rfm_score DESC
)
SELECT
	COUNT(order_id) AS num_customers,
    CASE 
        WHEN Rfm_score IN ('444', '445', '454', '455', '544', '545', '554', '555') THEN 'VIP Customers'
        WHEN Rfm_score IN ('114', '115', '124', '125', '214', '215', '224', '225') THEN 'At Risk'
        WHEN Rfm_score IN ('411', '412', '421', '422', '511', '512', '521', '522') THEN 'New Customers'
        WHEN Rfm_score IN ('443', '453', '543', '553') THEN 'Loyal Customers'
        WHEN Rfm_score IN ('111', '112', '121', '122', '211', '212', '221', '222') THEN 'Lost'
        ELSE 'Other customer'
    END AS Segment
FROM 
    Rfm_scores
GROUP BY 2
ORDER BY 1 DESC;
```
#### Analyzing which category is the most popular for each of the customers segments
```sql
WITH Rfm AS (
    SELECT
		od.category,
        lo.customer_name,
        od.order_id AS order_id,
        AGE(CURRENT_DATE, MAX(lo.order_date)) AS Recency,
        COUNT(od.order_id) AS Frequency,
        SUM(od.amount) AS Monetary,
        NTILE(5) OVER (ORDER BY AGE(CURRENT_DATE, MAX(lo.order_date)) DESC) AS R,
        NTILE(5) OVER (ORDER BY COUNT(od.order_id) ASC) AS F,
        NTILE(5) OVER (ORDER BY SUM(od.amount) ASC) AS M
    FROM 
        order_details od
    INNER JOIN 
        list_orders lo ON od.order_id = lo.order_id
    GROUP BY 
        1,2,3
),
Rfm_scores AS (
    SELECT 
		category,
		order_id,
        customer_name,
        CONCAT(R, F, M) AS Rfm_score
    FROM 
        Rfm 
    ORDER BY 
        Rfm_score DESC
)
SELECT
	category,
	COUNT(DISTINCT order_id) AS num_purchases,
    CASE 
        WHEN Rfm_score IN ('444', '445', '454', '455', '544', '545', '554', '555') THEN 'VIP Customers'
        WHEN Rfm_score IN ('114', '115', '124', '125', '214', '215', '224', '225') THEN 'At Risk'
        WHEN Rfm_score IN ('411', '412', '421', '422', '511', '512', '521', '522') THEN 'New Customers'
        WHEN Rfm_score IN ('443', '453', '543', '553') THEN 'Loyal Customers'
        WHEN Rfm_score IN ('111', '112', '121', '122', '211', '212', '221', '222') THEN 'Lost'
        ELSE 'Other customer'
    END AS Segment
FROM 
    Rfm_scores
GROUP BY 1,3
ORDER BY 3 DESC;
```
#### Most profitable Segments
```sql
WITH Rfm AS (
    SELECT
		od.profit,
		od.category,
        lo.customer_name,
        od.order_id AS order_id,
        AGE(CURRENT_DATE, MAX(lo.order_date)) AS Recency,
        COUNT(od.order_id) AS Frequency,
        SUM(od.amount) AS Monetary,
        NTILE(5) OVER (ORDER BY AGE(CURRENT_DATE, MAX(lo.order_date)) DESC) AS R,
        NTILE(5) OVER (ORDER BY COUNT(od.order_id) ASC) AS F,
        NTILE(5) OVER (ORDER BY SUM(od.amount) ASC) AS M
    FROM 
        order_details od
    INNER JOIN 
        list_orders lo ON od.order_id = lo.order_id
    GROUP BY 
        1,2,3,4
),
Rfm_scores AS (
    SELECT
		profit,
		category,
		order_id,
        customer_name,
        CONCAT(R, F, M) AS Rfm_score
    FROM 
        Rfm 
    ORDER BY 
        Rfm_score DESC
)
SELECT
	SUM(profit) AS profit,
    CASE 
        WHEN Rfm_score IN ('444', '445', '454', '455', '544', '545', '554', '555') THEN 'VIP Customers'
        WHEN Rfm_score IN ('114', '115', '124', '125', '214', '215', '224', '225') THEN 'At Risk'
        WHEN Rfm_score IN ('411', '412', '421', '422', '511', '512', '521', '522') THEN 'New Customers'
        WHEN Rfm_score IN ('443', '453', '543', '553') THEN 'Loyal Customers'
        WHEN Rfm_score IN ('111', '112', '121', '122', '211', '212', '221', '222') THEN 'Lost'
        ELSE 'Regular Customer'
    END AS Segment
FROM 
    Rfm_scores
GROUP BY 2
ORDER BY 1 DESC;
```
## Profit by Customer Segment

| Customer Segment   | Total Profit |
|--------------------|-------------|
| VIP Customers      | 19265.00    |
| Regular Customer   | 3919.00     |
| At Risk            | 2166.00     |
| New Customers      | 1776.00     |
| Loyal Customers    | 72.00       |
| Lost               | -3463.00    |

## Recommendations and Findings:

Based on the previous analysis, here are several recommendations for future decision-making:

- VIP Customers: VIP Customers are the most profitable segment in the company. Their most frequently purchased category is clothing, followed by electronics, with just five fewer purchases. Offering discounts and free clothing gifts with the purchase of an electronic item could be an effective strategy for retaining these valuable customers.

- Seasonal Profitability: The most profitable season is Spring with $13,403. In contrast, the least profitable seasons are Monsoon, with a loss of $13,249, and Summer, with a loss of $6,957.

- Retention Programs: Establish retention programs for at-risk clients by focusing on personalized marketing campaigns. For these customers, the most popular category is furniture.

- Regular Customers: Regular customers are a valuable part of the company, being the second most profitable segment. More feedback and engagement are necessary to record more buying patterns and convert them into loyal customers.