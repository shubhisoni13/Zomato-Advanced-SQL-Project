# Zomato-Advanced-SQL-Project

### 1. Create Database and Use It
 ```sql
-- Create the "zomato" database and switch to it. 
CREATE DATABASE zomato;
USE zomato;
```

### 2. View All Data from Tables
```sql
-- Retrieve all records from the primary tables in the database.
SELECT * FROM customers;
SELECT * FROM deliveries;
SELECT * FROM orders;
SELECT * FROM restaurants;
SELECT * FROM riders;
```

### 3. Count Null Values in Customers Table
```sql
-- Find the number of customers with missing Customer IDs or Registration Dates.
SELECT COUNT(*) FROM customers WHERE Customer_id IS NULL OR Registration_Date IS NULL;
```

### 4. Count Null Values in Deliveries Table
```sql
-- Find the number of deliveries with missing critical information like Order ID or Rider ID.
SELECT COUNT(*) FROM deliveries WHERE Order_ID IS NULL OR delivery_status IS NULL OR delivery_time IS NULL OR rider_id IS NULL;
```

### 5. Count Null Values in Orders Table
```sql
-- Identify orders with missing details such as Customer ID, Restaurant ID, or Amount.
SELECT COUNT(*) FROM orders WHERE Order_ID IS NULL OR Customer_ID IS NULL OR Restaurant_ID IS NULL OR Order_Items IS NULL OR Order_Date IS NULL OR Order_Time IS NULL OR `Status` IS NULL OR amount IS NULL;
```

### 6. Frequent Customer Orders
```sql
-- Fetch customers and their order details for orders placed in the last year, sorted by total orders.
SELECT c.customer_id, c.customer_name, o.order_items AS dishes, COUNT(*) AS total_orders 
FROM orders AS o 
JOIN customers AS c ON c.customer_id = o.customer_id 
WHERE o.order_date >= CURDATE() - INTERVAL 1 YEAR 
GROUP BY c.customer_id, c.customer_name, o.order_items 
ORDER BY c.customer_id, total_orders DESC;
```

### 7. Time Slot Analysis for Orders
```sql
-- Group orders by hourly time slots to identify the busiest periods.
SELECT 
    CASE
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 0 AND 1 THEN '00:00 - 02:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 2 AND 3 THEN '02:00 - 04:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 4 AND 5 THEN '04:00 - 06:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 6 AND 7 THEN '06:00 - 08:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 8 AND 9 THEN '08:00 - 10:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 10 AND 11 THEN '10:00 - 12:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 12 AND 13 THEN '12:00 - 14:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 14 AND 15 THEN '14:00 - 16:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 16 AND 17 THEN '16:00 - 18:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 18 AND 19 THEN '18:00 - 20:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 20 AND 21 THEN '20:00 - 22:00'
        WHEN EXTRACT(HOUR FROM ORDER_TIME) BETWEEN 22 AND 23 THEN '22:00 - 00:00'
    END AS TIME_SLOT, 
    COUNT(ORDER_ID) AS ORDER_COUNT 
FROM orders 
GROUP BY TIME_SLOT 
ORDER BY ORDER_COUNT DESC;
```

### 8. Customer Average Order Value (AOV)
```sql
-- Calculate the average order value for customers who placed more than three orders.
SELECT C.CUSTOMER_NAME, AVG(O.AMOUNT) AS AOV 
FROM ORDERS AS O 
JOIN CUSTOMERS AS C ON C.CUSTOMER_ID = O.CUSTOMER_ID 
GROUP BY C.CUSTOMER_NAME 
HAVING COUNT(O.ORDER_ID) > 3;
```

### 9. High-Value Customers
```sql
-- List customers who have spent more than 100K on food orders.
SELECT C.CUSTOMER_NAME, SUM(O.AMOUNT) AS TOTAL_SPENT 
FROM ORDERS AS O 
JOIN CUSTOMERS AS C ON C.CUSTOMER_ID = O.CUSTOMER_ID 
GROUP BY C.CUSTOMER_NAME 
HAVING SUM(O.AMOUNT) > 100000;
```

### 10. Undelivered Orders
```sql
-- Identify orders that were not delivered by nullifying invalid delivery IDs.
UPDATE DELIVERIES SET DELIVERY_ID = NULL WHERE TRIM(DELIVERY_ID) = '';
```

### 11. Restaurant Revenue Ranking
```sql
-- Rank restaurants by their total revenue within each city for the last year.
WITH RANKING_TABLE AS (
    SELECT 
        R.CITY, 
        R.RESTAURANT_NAME, 
        SUM(O.AMOUNT) AS REVENUE, 
        RANK() OVER (PARTITION BY R.CITY ORDER BY SUM(O.AMOUNT) DESC) AS RANK_
    FROM ORDERS AS O 
    JOIN RESTAURANTS AS R ON R.RESTAURANT_ID = O.RESTAURANT_ID 
    WHERE O.ORDER_DATE >= CURDATE() - INTERVAL 1 YEAR 
    GROUP BY R.CITY, R.RESTAURANT_NAME
)
SELECT * FROM RANKING_TABLE WHERE RANK_ = 1;
```

### 12. Customer Churn
```sql
-- Find customers who placed orders in 2020 but not in 2024.
SELECT DISTINCT CUSTOMER_ID 
FROM ORDERS 
WHERE EXTRACT(YEAR FROM STR_TO_DATE(ORDER_DATE, '%Y-%m-%d')) = 2020 
AND CUSTOMER_ID NOT IN (
    SELECT DISTINCT CUSTOMER_ID 
    FROM ORDERS 
    WHERE EXTRACT(YEAR FROM STR_TO_DATE(ORDER_DATE, '%Y-%m-%d')) = 2024
);
```

### 13. Rider Average Delivery Time
```sql
-- Calculate the average delivery time for each rider.
SELECT  
    D.RIDER_ID, 
    AVG(TIMESTAMPDIFF(MINUTE, O.ORDER_TIME, D.DELIVERY_TIME)) AS AVG_DELIVERY_TIME 
FROM ORDERS AS O 
JOIN DELIVERIES AS D ON O.ORDER_ID = D.ORDER_ID 
WHERE D.DELIVERY_STATUS = 'DELIVERED' 
GROUP BY D.RIDER_ID;
```

### 14. Monthly Growth Ratio of Restaurants
```sql
-- Compare the number of delivered orders each month to the previous month.
WITH growth_ratio AS (
    SELECT
        o.restaurant_id,
        DATE_FORMAT(STR_TO_DATE(o.order_date, '%Y-%m-%d'), '%Y-%m') AS month, 
        COUNT(o.order_id) AS current_month_orders,
        LAG(COUNT(o.order_id)) OVER (PARTITION BY o.restaurant_id ORDER BY DATE_FORMAT(STR_TO_DATE(o.order_date, '%Y-%m-%d'))) AS previous_month_orders
    FROM orders AS o
    JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'DELIVERED'
    GROUP BY o.restaurant_id, month
)
SELECT
    restaurant_id, 
    month, 
    previous_month_orders, 
    current_month_orders, 
    (current_month_orders / previous_month_orders) * 100 AS growth_ratio
FROM growth_ratio;
```

### 15. Rider Monthly Earnings
```sql
-- Calculate each rider's total monthly earnings, assuming they earn 8% of the order amount.
SELECT
    d.rider_id, 
    DATE_FORMAT(STR_TO_DATE(o.order_date, '%Y-%m-%d'), '%Y-%m') AS month, 
    SUM(o.amount) * 0.08 AS total_earnings
FROM orders AS o
JOIN deliveries AS d ON o.order_id = d.order_id
GROUP BY d.rider_id, month;
```

### 16. Order Frequency by Day
```sql
-- Identify the peak day of orders for each restaurant.
SELECT 
    r.restaurant_name, 
    DAYNAME(o.order_date) AS day, 
    COUNT(o.order_id) AS total_orders
FROM orders AS o
JOIN restaurants AS r ON o.restaurant_id = r.restaurant_id
GROUP BY r.restaurant_name, day
ORDER BY r.restaurant_name, total_orders DESC;
```

### 17. Customer Lifetime Value (CLV)
```sql
-- Calculate the total revenue generated by each customer.
SELECT 
    o.customer_id, 
    c.customer_name, 
    SUM(o.amount) AS CLV 
FROM orders AS o 
JOIN customers AS c ON o.customer_id = c.customer_id 
GROUP BY o.customer_id, c.customer_name;
```

### 18. Rider Efficiency
```sql
-- Evaluate rider efficiency by calculating average delivery times.
WITH delivery_times AS (
    SELECT 
        d.rider_id, 
        TIMESTAMPDIFF(MINUTE, o.order_time, d.delivery_time) AS delivery_time
    FROM orders AS o
    JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'DELIVERED'
)
SELECT 
    rider_id, 
    AVG(delivery_time) AS avg_delivery_time
FROM delivery_times
GROUP BY rider_id
ORDER BY avg_delivery_time;
```

### 19. Order Item Popularity
```sql
-- Track the popularity of specific order items over different seasons.
SELECT 
    order_items, 
    CASE
        WHEN EXTRACT(MONTH FROM order_date) BETWEEN 3 AND 5 THEN 'Spring'
        WHEN EXTRACT(MONTH FROM order_date) BETWEEN 6 AND 8 THEN 'Summer'
        WHEN EXTRACT(MONTH FROM order_date) BETWEEN 9 AND 11 THEN 'Autumn'
        ELSE 'Winter'
    END AS season,
    COUNT(order_items) AS total_orders
FROM orders
GROUP BY order_items, season
ORDER BY total_orders DESC;
```

### 20. City Revenue Ranking
```sql
-- Rank each city based on total revenue for the year 2023.
SELECT
    r.city, 
    SUM(o.amount) AS total_revenue, 
    RANK() OVER (ORDER BY SUM(o.amount) DESC) AS city_rank
FROM orders AS o
JOIN restaurants AS r ON o.restaurant_id = r.restaurant_id;
```

