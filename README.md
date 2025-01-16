---

# **Amazon USA Sales Analysis Project**

---

## **Project Overview**

I have worked on analyzing a 9 dependent datasets of over 20,000 sales records of Amazon-like e-commerce platform. This project involves extensive querying of customer behavior, product performance, and sales trends using PostgreSQL. Through this project, I have tackled various SQL problems, including revenue analysis, customer segmentation, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

---

### **Schema Structure**

```sql
CREATE TABLE category
(
category_id	INT PRIMARY KEY,
category_name VARCHAR(20)
);


CREATE TABLE customers
(
customer_id INT PRIMARY KEY,	
first_name	VARCHAR(20),
last_name	VARCHAR(20),
state VARCHAR(20)
);


CREATE TABLE sellers
(
seller_id INT PRIMARY KEY,
seller_name	VARCHAR(25),
origin VARCHAR(10)
);


CREATE TABLE products
(
product_id INT PRIMARY KEY,	
product_name VARCHAR(50),	
price	FLOAT,
cogs	FLOAT,
category_id INT, -- FK 
CONSTRAINT product_fk_category FOREIGN KEY(category_id) REFERENCES category(category_id)
);


CREATE TABLE orders
(
order_id INT PRIMARY KEY, 	
order_date	DATE,
customer_id	INT, -- FK
seller_id INT, -- FK 
order_status VARCHAR(15),
CONSTRAINT orders_fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
CONSTRAINT orders_fk_sellers FOREIGN KEY (seller_id) REFERENCES sellers(seller_id)
);

CREATE TABLE order_items
(
order_item_id INT PRIMARY KEY,
order_id INT,	-- FK 
product_id INT, -- FK
quantity INT,	
price_per_unit FLOAT,
CONSTRAINT order_items_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id),
CONSTRAINT order_items_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
);


CREATE TABLE payments
(
payment_id	
INT PRIMARY KEY,
order_id INT, -- FK 	
payment_date DATE,
payment_status VARCHAR(20),
CONSTRAINT payments_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);


CREATE TABLE shippings
(
shipping_id	INT PRIMARY KEY,
order_id	INT, -- FK
shipping_date DATE,	
return_date	 DATE,
shipping_providers	VARCHAR(15),
delivery_status VARCHAR(15),
CONSTRAINT shippings_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);


CREATE TABLE inventory
(
inventory_id INT PRIMARY KEY,
product_id INT, -- FK
stock INT,
warehouse_id INT,
last_stock_date DATE,
CONSTRAINT inventory_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

---

## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer behavior
- Sales trends
- Payment and shipping analysis
- Product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.

---

## **Solving Business Problems**

### Solutions Implemented:
1. Top Selling Products
Query the top 10 products by total sales value.
Challenge: Include product name, total quantity sold, and total sales value.

```sql
ALTER TABLE order_items
ADD COLUMN total_sales NUMERIC;

UPDATE order_items
SET total_sales= quantity*price_per_unit;
	
SELECT p.product_name, COUNT(o.order_id) as units_sold, ROUND(SUM(oi.total_sales)::NUMERIC,0) as total_sales
FROM
products AS p
JOIN
order_items AS oi
	ON p.product_id=oi.product_id
JOIN
orders AS o
	ON oi.order_id=o.order_id
GROUP BY 1
ORDER BY 3 DESC LIMIT 10;
```

2. Revenue by Category
Calculate total revenue generated by each product category.
Challenge: Include the percentage contribution of each category to total revenue.

```sql
select p.category_id, c.category_name, ROUND(SUM(oi.total_sales)::NUMERIC,0) AS total_revenue,
	(SUM(oi.total_sales)*100)/(SELECT SUM(total_sales) FROM order_items) AS percentage_contri
FROM 
category as c
JOIN
products as p
	ON c.category_id=p.category_id
JOIN
order_items as oi
	ON oi.product_id=p.product_id
GROUP BY 1,2
ORDER BY 3 DESC;
```

3. Average Order Value (AOV)
Compute the average order value for each customer.
Challenge: Include only customers with more than 5 orders.

```sql
select c.customer_id, CONCAT(c.first_name,' ',c.last_name) as name, COUNT(o.order_id) as total_orders,
	ROUND(SUM(oi.total_sales)::NUMERIC/COUNT(o.order_id),0) AS AOV
FROM customers AS c
JOIN
orders AS o
	ON c.customer_id=o.customer_id
JOIN
order_items AS oi
	ON o.order_id=oi.order_id
GROUP BY 1,2
HAVING COUNT(o.order_id)>5;
```

4. Monthly Sales Trend
Query monthly total sales over the past year.
Challenge: Display the sales trend, grouping by month, return monthly sales.

```sql
SELECT EXTRACT(MONTH FROM o.order_date) as month, ROUND(SUM(oi.total_sales)::NUMERIC,2) as monthly_sales
FROM 
order_items as oi
join
orders as o
	ON oi.order_id=o.order_id
	WHERE EXTRACT(YEAR FROM o.order_date)= EXTRACT(YEAR FROM CURRENT_DATE)-1
group by 1;
```


5. Customers with No Purchases
Find customers who have registered but never placed an order.
Challenge: List customer details and the time since their registration.

```sql
SELECT * FROM customers
WHERE customer_id NOT IN (SELECT DISTINCT customer_id from orders);
```


6. Least-Selling Categories by State
Identify the least-selling product category for each state.
Challenge: Include the total sales for that category within each state.

```sql
WITH ranking_table AS (
select c.category_name, cu.state, ROUND(SUM(oi.total_sales)::numeric,0) as least_sales,
	rank() over (partition by cu.state ORDER BY SUM(oi.total_sales)) as rank
	FROM
category AS c
join
products AS p
	ON c.category_id=p.category_id
join
order_items as oi
	ON oi.product_id=p.product_id
join
orders as o
	ON o.order_id=oi.order_id
join
customers as cu
	ON cu.customer_id=o.customer_id
GROUP BY 1,2
)
SELECT * FROM ranking_table
where rank=1;
```


7. Customer Lifetime Value (CLTV)
Calculate the total value of orders placed by each customer over their lifetime.
Challenge: Rank customers based on their CLTV.
```sql
SELECT cu.customer_id, CONCAT(cu.first_name,' ',cu.last_name) as customer_name, ROUND(sum(total_sales)::numeric,0) as CLTV
FROM
customers as cu
join
orders as o
	ON cu.customer_id=o.customer_id
join
order_items as oi
	ON o.order_id=oi.order_id
GROUP BY 1
ORDER BY 3 DESC;
```


8. Inventory Stock Alerts
Query products with stock levels below a certain threshold (e.g., less than 10 units).
Challenge: Include last restock date and warehouse information.

```sql
select i.warehouse_id, i.inventory_id, p.product_name, i.stock as stocks_left, i.last_stock_date as last_restock
from
inventory as i
join
products as p
	ON i.product_id=p.product_id
WHERE i.stock<10;
```

9. Shipping Delays
Identify orders where the shipping date is later than 3 days after the order date.
Challenge: Include customer, order details, and delivery provider.

```sql
SELECT c.customer_id, CONCAT(c.first_name,' ',c.last_name) as customer_name, 
	c.state, o.order_id, o.order_date, o.seller_id, o.order_status, s.shipping_providers,
	s.shipping_date - o.order_date as days_took_to_ship
	FROM
customers as c
join
orders as o
	on o.customer_id=c.customer_id
join
shippings as s
	on s.order_id=o.order_id
WHERE s.shipping_date - o.order_date >3;
```

10. Payment Success Rate 
Calculate the percentage of types of payments_status across all orders.
Challenge: Include breakdowns by payment status (e.g., failed, pending).

```sql
select p.payment_status, count(o.order_id) as orders_count,
	CONCAT(count(o.order_id)*100/(select count(order_id) from orders),'%') as contribution
	from
payments as p
join
orders as o
	on p.order_id=o.order_id
GROUP BY payment_status;
```

11. Top Performing Sellers
Find the top 5 sellers based on total sales value.
Challenge: Include both completed and cancelled orders, and display the percentage of successful orders.

```sql
select *,CONCAT((orders_completed)*100/(orders_completed+orders_cancelled),'%') as successful_perc 
	from (
SELECT s.seller_id as seller_id, s.seller_name as seller_name, ROUND(SUM(oi.total_sales)::numeric,0) as total_sales,
	SUM(case when o.order_status='Completed' then 1 else 0 end) as orders_completed,
	SUM(case when o.order_status='Cancelled' then 1 else 0 end) as orders_cancelled
from	
sellers as s
join
orders as o
	on s.seller_id=o.seller_id
join
order_items as oi
	on o.order_id=oi.order_id
group by 1,2
order by 3 desc limit 5
);
```


12. Product Profit Margin
Calculate the profit margin for each product (difference between price and cost of goods sold).
Challenge: Rank products by their profit margin, showing highest to lowest.
*/


```sql
--Profit Margin is Profit*100/Revenue

SELECT p.product_name,p.cogs, oi.price_per_unit, oi.quantity as qty_sold, oi.total_sales as sales,
	ROUND(((oi.total_sales::numeric - (p.cogs * oi.quantity)::numeric) / (oi.total_sales)::numeric) * 100, 0) AS profit_margin

from
products as p
join
order_items as oi
on
	p.product_id=oi.product_id
group by 1,2,3,4,5
order by 6 desc;
```

13. Most Returned Products
Query the top 10 products by the number of returns.
Challenge: Display the return rate as a percentage of total units sold for each product.

```sql
select p.product_id, p.product_name,
	sum(case when o.order_status='Returned' then 1 else 0 end) as returned,
	round((sum(case when o.order_status='Returned' then 1 else 0 end)::numeric)*100/(count(o.order_id)),0) as return_percentage
from
products as p
join
order_items as oi
	on p.product_id=oi.product_id
join
orders as o
	on oi.order_id=o.order_id
group by 1,2
order by 4 desc
```

14. Orders Pending Shipment
Find orders that have been paid but are still not delivered.
Challenge: Include order details, payment date, and customer information.

```sql
select cu.customer_id, p.payment_id, p.order_id, CONCAT(cu.first_name,' ',cu.last_name) as customer_name, cu.state, p.payment_status, 
	p.payment_date, s.delivery_status
from
customers as cu
join
orders as o
	on cu.customer_id=o.customer_id
join
payments as p
	on o.order_id=p.order_id
join
shippings as s
	on s.order_id=p.order_id
where p.payment_status= 'Payment Successed' AND (s.delivery_status!= 'Delivered');
```


15. Inactive Sellers
Identify sellers who haven’t made any sales in the last 6 months.

```sql
SELECT s.seller_id, s.seller_name
FROM sellers AS s
WHERE s.seller_id NOT IN (SELECT seller_id 
                          FROM orders
                          WHERE order_date > current_date - interval '6 months');
```


16. IDENTITY customers into returning or new
If the customer has done more than 5 returns, categorize them as returning, otherwise new.
Challenge: List customers id, name, total orders, total returns.
```sql
SELECT customer_id, customer_name, total_orders, (CASE WHEN total_returns>5 THEN 'returning_customers' ELSE 'new' END) as category
FROM
(
SELECT c.customer_id as customer_id, CONCAT(c.first_name,' ', c.last_name) as customer_name, COUNT(oi.order_id) as total_orders,
	SUM(CASE WHEN o.order_status= 'Returned' THEN 1 ELSE 0 END) as total_returns 
FROM
customers AS c
JOIN
orders AS o
	ON c.customer_id=o.customer_id
JOIN
order_items AS oi
	ON oi.order_id=o.order_id
GROUP BY 1
);
```

17. Top 5 Customers by Orders in Each State
Identify the top 5 customers with the highest number of orders for each state.
Challenge: Include the number of orders and total sales for each customer.

```sql
SELECT * FROM
(
	SELECT c.customer_id, c.state, CONCAT(c.first_name,' ', c.last_name) as customer_name, COUNT(o.order_id) as total_orders, 
	ROUND(SUM(oi.total_sales)::NUMERIC,0) as total_sales,
	ROW_NUMBER() OVER (PARTITION BY c.state ORDER BY COUNT(o.order_id) DESC) as rank
FROM
customers AS c
JOIN
orders AS o
	ON c.customer_id=o.customer_id
JOIN
order_items AS oi
	ON oi.order_id=o.order_id
GROUP BY 1,2,3 
)
WHERE RANK <= 5
```

18. Revenue by Shipping Provider
Calculate the total revenue handled by each shipping provider.
Challenge: Include the total number of orders handled.

```sql
SELECT s.shipping_providers, ROUND(SUM(oi.total_sales)::NUMERIC,2) as total_sales, COUNT(s.order_id) as total_orders
FROM
orders AS o
JOIN
order_items AS oi
	ON oi.order_id=o.order_id
JOIN
shippings as s
	ON oi.order_id=s.order_id
GROUP BY 1 
ORDER BY 2 DESC;
```


19. Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result
Note: Decrease ratio = cs-ls/ls* 100 (cs = current_year ls=last_year)

```SQL
WITH last_year_sale
as
(
SELECT 
	p.product_id,
	p.product_name,
	SUM(oi.total_sales) as revenue
FROM orders as o
JOIN 
order_items as oi
ON oi.order_id = o.order_id
JOIN 
products as p
ON 
p.product_id = oi.product_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2022
GROUP BY 1, 2
),

current_year_sale
AS
(
SELECT 
	p.product_id,
	p.product_name,
	SUM(oi.total_sales) as revenue
FROM orders as o
JOIN 
order_items as oi
ON oi.order_id = o.order_id
JOIN 
products as p
ON 
p.product_id = oi.product_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2023
GROUP BY 1, 2
)

SELECT
	cs.product_id,
	ROUND(ls.revenue::numeric,0) as last_year_revenue,
	ROUND(cs.revenue::numeric,0) as current_year_revenue,
	ROUND((ls.revenue - cs.revenue)::numeric,0) as rev_diff,
	ROUND((cs.revenue - ls.revenue)::numeric/ls.revenue::numeric * 100, 0) as revenue_dec_ratio
FROM last_year_sale as ls
JOIN
current_year_sale as cs
ON ls.product_id = cs.product_id
WHERE 
	ls.revenue > cs.revenue
ORDER BY 5 
LIMIT 10;
```


20. Date Conversion
Convert the order_date to a Indian date format (e.g., 'DD-MM-YYYY')
Challenge: Display it with the order ID.

```SQL
SELECT order_id, TO_CHAR(order_date, 'DD-MM-YYYY') AS indian_formatted_date
FROM orders;
```

---

## **Learning Outcomes**

This project enabled me to:
- Design and implement a normalized database schema.
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---

## **Conclusion**

This SQL project successfully demonstrates my ability to solve real-world e-commerce problems using structured queries. From improving customer retention to optimizing inventory and logistics, the project provides valuable insights into operational challenges and solutions.

By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems and drive business decision-making.

---
