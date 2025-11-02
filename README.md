# Food-Delivery-SQL-Analytics-PostgreSQL-

---
# 🍽️ Food Delivery SQL Analytics (PostgreSQL)

## 🧭 Project Overview

This project contains **20 advanced PostgreSQL SQL queries** designed to analyze a food delivery business dataset.
It focuses on real-world analytics scenarios such as:

* Customer behavior analysis and segmentation
* Restaurant performance and revenue ranking
* Rider efficiency and ratings
* Time-based sales trends
* Warranty claims and product insights

The project demonstrates mastery of **joins, aggregations, window functions, CTEs, subqueries, and date/time functions**.

---

## 🛠️ Tech Stack

* **Database:** PostgreSQL
* **Concepts Used:** Joins, Aggregations, Window Functions (RANK, DENSE_RANK, LAG), CTEs, Subqueries, Date/Time Functions, Ranking and Percentile Calculations

---

## 🗂️ Database Schema & ERD

**Tables:**

1. `CATEGORY` – Product categories
2. `PRODUCTS` – Details of products
3. `SALES` – Sales transactions
4. `STORES` – Store information
5. `WARRANTY` – Warranty claims for sales

**ASCII ERD Diagram:**

```
CATEGORY
---------
category_id PK
category_name

PRODUCTS
--------
product_id PK
product_name
category_id FK -> CATEGORY(category_id)
price
launch_date

STORES
------
store_id PK
store_name
country
city

SALES
-----
sale_id PK
product_id FK -> PRODUCTS(product_id)
store_id FK -> STORES(store_id)
quantity
total_price
sale_date

WARRANTY
--------
claim_id PK
sale_id FK -> SALES(sale_id)
claim_date
repair_status
---
```



```

---

## 📊 Queries (Sequential 1–20)

### 1. Top 5 Dishes Ordered by Customer "Arjun Mehta" in Last 1 Year

```sql
SELECT
    c.customer_id,
    c.customer_name,
    o.order_item AS dishes,
    COUNT(*) AS total_orders
FROM orders AS o
JOIN customers AS c
    ON c.customer_id = o.customer_id
WHERE
    o.order_date >= CURRENT_DATE - INTERVAL '1 year'
    AND c.customer_name = 'Arjun Mehta'
GROUP BY 1,2,3
ORDER BY total_orders DESC
LIMIT 5;
```

*Identifies the most frequently ordered dishes by a specific customer over the last year.*

---

### 2. Popular Time Slots (2-Hour Intervals)

```sql
SELECT
    FLOOR(EXTRACT(HOUR FROM order_time) / 2) * 2 AS start_hour,
    FLOOR(EXTRACT(HOUR FROM order_time) / 2) * 2 + 2 AS end_hour,
    CONCAT(FLOOR(EXTRACT(HOUR FROM order_time) / 2) * 2, '-', FLOOR(EXTRACT(HOUR FROM order_time) / 2) * 2 + 2) AS time_slot,
    COUNT(*) AS total_orders
FROM orders
GROUP BY 1,2,3
ORDER BY total_orders DESC;
```

*Finds the 2-hour time intervals with the most orders.*

---

### 3. Average Order Value per Customer (>750 Orders)

```sql
SELECT
    o.customer_id,
    c.customer_name,
    AVG(o.total_amount) AS aov
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
GROUP BY 1,2
HAVING COUNT(order_id) > 750;
```

*Calculates AOV for high-frequency customers.*

---

### 4. High-Value Customers (Spent > 100K)

```sql
SELECT
    c.customer_id,
    c.customer_name,
    SUM(o.total_amount) AS total_spent
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
GROUP BY 1,2
HAVING SUM(o.total_amount) > 100000;
```

*Lists customers with total spending exceeding 100K.*

---

### 5. Orders Without Delivery

```sql
SELECT
    r.restaurant_name,
    COUNT(o.order_id) AS total_orders
FROM orders AS o
LEFT JOIN restaurants AS r ON r.restaurant_id = o.restaurant_id
LEFT JOIN deliveries AS d ON d.order_id = o.order_id
WHERE d.delivery_id IS NULL
GROUP BY 1
ORDER BY total_orders DESC;
```

*Identifies undelivered orders per restaurant.*

---

### 6. Restaurant Revenue Ranking

```sql
WITH RANKING_TABLE AS (
    SELECT
        r.restaurant_name,
        r.city,
        SUM(o.total_amount) AS total_revenue,
        RANK() OVER (
            PARTITION BY r.city
            ORDER BY SUM(o.total_amount) DESC
        ) AS rank
    FROM restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY r.restaurant_name, r.city
)
SELECT *
FROM RANKING_TABLE
WHERE rank = 1;
```

*Ranks restaurants by total revenue within their city.*

---

### 7. Most Popular Dish by City

```sql
SELECT
    dish,
    city,
    total_order,
    rank
FROM (
    SELECT
        o.order_item AS dish,
        r.city,
        COUNT(*) AS total_order,
        RANK() OVER (
            PARTITION BY r.city
            ORDER BY COUNT(*) DESC
        ) AS rank
    FROM orders o
    JOIN restaurants r ON r.restaurant_id = o.restaurant_id
    GROUP BY o.order_item, r.city
) AS t
WHERE rank = 1;
```

*Identifies the most ordered dish per city.*

---

### 8. Customer Churn (2023 vs 2024)

```sql
WITH customer_2023 AS (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE EXTRACT(YEAR FROM order_date) = 2023
)
SELECT c.customer_id
FROM customer_2023 c
JOIN customers cu ON cu.customer_id = c.customer_id
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND EXTRACT(YEAR FROM o.order_date) = 2024
);
```

*Finds customers active in 2023 but inactive in 2024.*

---

### 9. Cancellation Rate Comparison (2023 vs 2024)

```sql
WITH cancellation_ratio_2023 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS not_delivered
    FROM orders o
    LEFT JOIN deliveries d ON o.order_id = d.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2023
    GROUP BY o.restaurant_id
),
cancellation_ratio_2024 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS not_delivered
    FROM orders o
    LEFT JOIN deliveries d ON o.order_id = d.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2024
    GROUP BY o.restaurant_id
),
last_year_data AS (
    SELECT restaurant_id,
           ROUND((not_delivered::NUMERIC / NULLIF(total_orders,0)) * 100,2) AS cancel_ratio
    FROM cancellation_ratio_2023
),
current_year_data AS (
    SELECT restaurant_id,
           ROUND((not_delivered::NUMERIC / NULLIF(total_orders,0)) * 100,2) AS cancel_ratio
    FROM cancellation_ratio_2024
)
SELECT 
    l.restaurant_id,
    l.cancel_ratio AS last_year_cancel_ratio,
    c.cancel_ratio AS current_year_cancel_ratio,
    ROUND(COALESCE(c.cancel_ratio,0) - COALESCE(l.cancel_ratio,0),2) AS rate_difference
FROM last_year_data l
LEFT JOIN current_year_data c ON l.restaurant_id = c.restaurant_id
ORDER BY rate_difference DESC;
```

*Compares cancellation rates for restaurants between two years.*

---

### 10. Rider Average Delivery Time

```sql
SELECT 
    o.order_id,
    o.order_time,
    d.rider_id,
    d.delivery_time - o.order_time AS time_difference,
    EXTRACT(EPOCH FROM (d.delivery_time - o.order_time)) / 60 AS time_difference_in_min
FROM orders o
JOIN deliveries d ON o.order_id = d.order_id
WHERE d.delivery_status = 'Delivered';
```

*Calculates rider delivery time in minutes.*

---

### 11. Monthly Restaurant Growth Ratio

```sql
WITH GROWTH_RATIO AS (
    SELECT  
        r.restaurant_name,
        TO_CHAR(o.order_date,'YYYY-MM') AS month,
        COUNT(o.order_id) AS current_month_order,
        LAG(COUNT(o.order_id)) OVER (
            PARTITION BY r.restaurant_name
            ORDER BY TO_CHAR(o.order_date,'YYYY-MM')
        ) AS previous_month_order
    FROM restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    JOIN deliveries d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'Delivered'
    GROUP BY r.restaurant_name, TO_CHAR(o.order_date,'YYYY-MM')
)
SELECT 
    restaurant_name,
    month,
    current_month_order,
    ROUND(((current_month_order::NUMERIC - previous_month_order::NUMERIC)/NULLIF(previous_month_order::NUMERIC,0))*100,2) AS growth_ratio
FROM GROWTH_RATIO
ORDER BY restaurant_name, month;
```

*Calculates month-over-month delivered order growth per restaurant.*

---

### 12. Customer Segmentation (Gold/Silver)

```sql
WITH INDIVIDUAL_SPENT AS (
    SELECT customer_id,
           COUNT(order_id) AS total_orders,
           SUM(total_amount) AS total_spent
    FROM orders
    GROUP BY customer_id
),
TOTAL_AVERAGE AS (
    SELECT AVG(total_amount) AS average_order_value FROM orders
),
CATEGORIZE AS (
    SELECT i.customer_id,
           i.total_orders,
           i.total_spent,
           CASE WHEN i.total_spent > t.average_order_value THEN 'GOLD' ELSE 'SILVER' END AS category
    FROM INDIVIDUAL_SPENT i
    CROSS JOIN TOTAL_AVERAGE t
)
SELECT category,
       SUM(total_orders) AS total_orders,
       SUM(total_spent) AS total_revenue
FROM CATEGORIZE
GROUP BY category
ORDER BY total_revenue DESC;
```

*Segments customers into Gold/Silver based on total spending vs average order value.*

---

### 13. Rider Monthly Earnings (8% Commission)

```sql
SELECT
    r.rider_id,
    TO_CHAR(o.order_date,'YYYY-MM') AS month,
    ROUND(SUM(o.total_amount*0.08),2) AS riders_earnings
FROM riders r
JOIN deliveries d ON r.rider_id = d.rider_id
JOIN orders o ON d.order_id = o.order_id
GROUP BY r.rider_id, TO_CHAR(o.order_date,'YYYY-MM')
ORDER BY r.rider_id, month;
```

*Calculates monthly earnings for riders.*

---

### 14. Rider Rating Analysis (3-5 Stars)

```sql
WITH TIME AS (
    SELECT d.rider_id,
           ROUND(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60,2) AS time_taken
    FROM deliveries d
    JOIN orders o ON d.order_id = o.order_id
),
RATING AS (
    SELECT rider_id,
           CASE 
               WHEN time_taken < 15 THEN '5 STAR'
               WHEN time_taken BETWEEN 15 AND 20 THEN '4 STAR'
               ELSE '3 STAR'
           END AS star_rating
    FROM TIME
)
SELECT rider_id,
       COUNT(CASE WHEN star_rating='5 STAR' THEN 1 END) AS five_star_count,
       COUNT(CASE WHEN star_rating='4 STAR' THEN 1 END) AS four_star_count,
       COUNT(CASE WHEN star_rating='3 STAR' THEN 1 END)
```


AS three_star_count
FROM RATING
GROUP BY rider_id
ORDER BY rider_id;

````
*Assigns rider ratings based on delivery speed.*

---

### 15. Order Frequency by Day
```sql
SELECT restaurant_name,
       day,
       total_orders
FROM (
    SELECT r.restaurant_name,
           o.order_date,
           TO_CHAR(o.order_date,'DAY') AS day,
           COUNT(o.order_id) AS total_orders,
           RANK() OVER (PARTITION BY r.restaurant_name ORDER BY COUNT(o.order_id) DESC) AS rank
    FROM restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    GROUP BY r.restaurant_name, o.order_date
) AS t
WHERE rank=1;
````

*Identifies peak order days per restaurant.*

---

### 16. Customer Lifetime Value

```sql
SELECT customer_id,
       SUM(total_amount) AS customers_lifetime_value
FROM orders
GROUP BY customer_id
ORDER BY SUM(total_amount) DESC;
```

*Calculates total revenue per customer.*

---

### 17. Monthly Sales Trends

```sql
SELECT TO_CHAR(order_date,'YYYY-MM') AS month,
       SUM(total_amount) AS total_sale,
       LAG(SUM(total_amount),1) OVER (ORDER BY TO_CHAR(order_date,'YYYY-MM')) AS previous_month_sale
FROM orders
GROUP BY 1;
```

*Compares each month’s total sales to the previous month.*

---

### 18. Rider Efficiency (Average Delivery Time)

```sql
WITH DELIVERY_TABLE AS (
    SELECT d.rider_id,
           EXTRACT(EPOCH FROM (d.delivery_time - o.order_time + CASE WHEN o.order_time>d.delivery_time THEN INTERVAL '1 day' ELSE INTERVAL '0 day' END))/60 AS time_taken
    FROM orders o
    JOIN deliveries d ON o.order_id=d.order_id
    WHERE d.delivery_status='Delivered'
),
RIDERS_TIME AS (
    SELECT rider_id,
           AVG(time_taken) AS average
    FROM DELIVERY_TABLE
    GROUP BY rider_id
)
SELECT MIN(average), MAX(average) FROM RIDERS_TIME;
```

*Identifies riders with fastest and slowest average delivery times.*

---

### 19. Order Item Popularity by Season

```sql
SELECT order_item,
       season,
       COUNT(*) AS total_orders
FROM (
    SELECT order_item,
           TO_CHAR(order_date,'YYYY-MM') AS month,
           CASE 
               WHEN EXTRACT(MONTH FROM order_date) BETWEEN 3 AND 5 THEN 'SPRING'
               WHEN EXTRACT(MONTH FROM order_date) BETWEEN 6 AND 8 THEN 'SUMMER'
               WHEN EXTRACT(MONTH FROM order_date) BETWEEN 9 AND 11 THEN 'AUTUMN'
               ELSE 'WINTER'
           END AS season
    FROM orders
) AS season_data
GROUP BY order_item, season
ORDER BY total_orders DESC;
```

*Tracks seasonal demand trends for menu items.*

---

### 20. Monthly Restaurant Growth Ratio (Delivered Orders)

```sql
WITH FIRST_ORDERS AS (
    SELECT o.restaurant_id,
           MIN(o.order_date) AS first_order
    FROM orders o
    GROUP BY o.restaurant_id
),
ORDER_COUNTS AS (
    SELECT r.restaurant_name,
           EXTRACT(YEAR FROM o.order_date) AS year,
           COUNT(d.order_id) AS total_order
    FROM deliveries d
    JOIN orders o ON d.order_id=o.order_id
    JOIN first_orders fr ON o.restaurant_id=fr.restaurant_id AND o.order_date >= fr.first_order
    JOIN restaurants r ON o.restaurant_id=r.restaurant_id
    WHERE d.delivery_status='Delivered'
    GROUP BY r.restaurant_name, EXTRACT(YEAR FROM o.order_date)
),
ORDER_ANALYSIS AS (
    SELECT restaurant_name,
           year,
           total_order,
           LAG(total_order,1) OVER (PARTITION BY restaurant_name ORDER BY year) AS previous_order_count
    FROM ORDER_COUNTS
)
SELECT restaurant_name,
       year,
       total_order,
       previous_order_count,
       ROUND(((total_order - COALESCE(previous_order_count,0))/NULLIF(previous_order_count,0))*100,2) AS growth_ratio
FROM ORDER_ANALYSIS
ORDER BY restaurant_name, year;
```

*Calculates year-over-year growth of delivered orders per restaurant.*


---

## 👨‍💻 Author

**Md. Sarwar Hossen Sagar**
📧 [sarwarhsagar@gmail.com](mailto:sarwarhsagar@gmail.com)
🔗 [Portfolio](https://www.datascienceportfol.io/sarwarhsagar)

---




