# 🍜 Case Study Marketing Analytics

## Solution

### 1. What is the total amount each customer spent at the restaurant?

````sql
SELECT 
s.customer_id, 
SUM(m.price) AS price
FROM dannys_diner.sales s
INNER JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY price DESC;
````
![image](https://user-images.githubusercontent.com/104872221/170370018-14cda9a2-0c90-4de0-8d4c-9d6547e270d4.png)

### 2. How many days has each customer visited the restaurant?

````sql
SELECT 
	customer_id, COUNT(DISTINCT order_date) AS customer_visits
FROM dannys_diner.sales
GROUP BY customer_id;
````

![image](https://user-images.githubusercontent.com/104872221/170375304-0c206419-327a-4688-b421-8a15451f4a55.png)



### 3. What was the first item from the menu purchased by each customer?

````sql
WITH cte_rnk AS (
SELECT 
*,
DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY order_date) AS rnk
FROM dannys_diner.sales
)

SELECT 
customer_id, order_date, product_id 
FROM cte_rnk
WHERE rnk = 1;
````

![image](https://user-images.githubusercontent.com/104872221/170370596-024e508b-8015-4f4a-b0ce-af98bade5fc6.png)


### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

````sql
SELECT 
menu.product_name, COUNT(sales.product_id) as order_count	
FROM dannys_diner.sales AS sales
INNER JOIN dannys_diner.menu AS menu
ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY order_count DESC;
````

![image](https://user-images.githubusercontent.com/104872221/170370817-3604a22c-6050-46fe-b9b0-ddd0c16efcbc.png)


### 5. Which item was the most popular for each customer?

````sql
WITH cte_popular AS (
SELECT
	sales.customer_id, 
	menu.product_name, 
	COUNT(sales.product_id) AS order_count,
	DENSE_RANK() OVER (PARTITION BY sales.customer_id ORDER BY COUNT(sales.customer_id) DESC) AS rank
FROM dannys_diner.sales AS sales
INNER JOIN dannys_diner.menu AS menu
ON sales.product_id = menu.product_id
GROUP BY sales.customer_id, menu.product_name
)
SELECT 
	customer_id,
	product_name,
	order_count
FROM cte_popular
WHERE rank = 1;
````

![image](https://user-images.githubusercontent.com/104872221/170371154-dc115055-ff53-4522-9b86-7b41d96d9338.png)


### 6. Which item was purchased first by the customer after they became a member?

````sql
WITH cte_member_orders AS (
SELECT 
	s.customer_id,
	s.order_date,
	s.product_id,
	mm.join_date,
	DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY order_date) AS rank	
FROM dannys_diner.members mm
INNER JOIN dannys_diner.sales s
ON s.customer_id = mm.customer_id
WHERE s.order_date >= mm.join_date
)

SELECT 
	m1.customer_id,
	m1.order_date,
	m2.product_name
FROM cte_member_orders AS m1
INNER JOIN dannys_diner.menu AS m2
ON m1.product_id = m2.product_id
WHERE rank = 1;
````

![image](https://user-images.githubusercontent.com/104872221/170371722-549e2bd3-81da-41ed-bffe-a604e82c657e.png)


### 7. Which item was purchased just before the customer became a member?

````sql
WITH cte_before_membership AS (
SELECT
	s.customer_id,
	s.order_date,
	s.product_id,
	m1.join_date,
	m2.product_name,
	DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS rank
FROM dannys_diner.members AS m1
INNER JOIN dannys_diner.sales AS s
ON m1.customer_id = s.customer_id
INNER JOIN dannys_diner.menu AS m2
ON s.product_id = m2.product_id
WHERE s.order_date < m1.join_date
)

SELECT 
	customer_id,
	product_name,
	order_date
FROM cte_before_membership
WHERE rank = 1;
````


![image](https://user-images.githubusercontent.com/104872221/170375827-67bc6595-4cb9-4242-9702-0594152ab268.png)



### 8. What is the total items and amount spent for each member before they became a member?

````sql
WITH cte_before_membership AS (
SELECT
	s.customer_id,
	s.product_id,
	m2.price
FROM dannys_diner.members AS m1
INNER JOIN dannys_diner.sales AS s
ON m1.customer_id = s.customer_id
INNER JOIN dannys_diner.menu AS m2
ON s.product_id = m2.product_id
WHERE s.order_date < m1.join_date
)

SELECT 
	customer_id,
	COUNT(DISTINCT product_id) as total_items,
	SUM(price) AS total_spent
FROM cte_before_membership
GROUP BY customer_id;
````

![image](https://user-images.githubusercontent.com/104872221/170372740-dd6e8e30-cec1-4b8d-9b60-a3025481e09a.png)


### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

````sql
WITH cte_point AS (
SELECT
	s.customer_id,
	mm.product_name,
	SUM(mm.price) AS total_spent
FROM dannys_diner.sales AS s
INNER JOIN dannys_diner.menu AS mm
ON mm.product_id = s.product_id
GROUP BY customer_id, product_name
)

SELECT
	customer_id,
	SUM(CASE WHEN product_name = 'sushi' THEN total_spent * 10 * 2
	ELSE total_spent * 10
	END ) AS total_points
FROM cte_point
GROUP BY customer_id;
````

![image](https://user-images.githubusercontent.com/104872221/170372809-c2b0b9d5-b193-45a5-98d7-4f44c3793cb4.png)



### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

````sql
WITH cte_dates AS
( SELECT
		*,
		DATEADD(DAY, 6, join_date) AS valid_date,
			EOMONTH('2021-01-31') AS last_date
	FROM dannys_diner.members AS m
)

SELECT
	d.customer_id,
	s.order_date,
	d.join_date,
	d.valid_date,
	d.last_date,
	m.product_name,
	m.price,
		SUM(
			CASE WHEN m.product_name = 'sushi' THEN 2*10*m.price
				WHEN s.order_date BETWEEN d.join_date AND d.valid_date THEN 2*10*m.price
				ELSE 10*m.price END) AS points
	FROM cte_dates AS d
	INNER JOIN dannys_diner.sales AS s
		ON s.customer_id = d.customer_id
	INNER JOIN dannys_diner.menu AS m
		ON s.product_id = m.product_id
	WHERE s.order_date < d.last_date
	GROUP BY  d.customer_id, s.order_date, d.join_date, d.valid_date, d.last_date, m.product_name, m.price;
  ````
![image](https://user-images.githubusercontent.com/104872221/170372871-47f5cc94-1fb5-4843-b94e-693cffeb1966.png)


### BONUS QUESTIONS

````sql
SELECT
	s.customer_id,
	s.order_date,
	m2.product_name,
	m2.price,
	(CASE WHEN s.order_date >= m1.join_date THEN 'Y'
	ELSE 'N'
	END) AS member
FROM dannys_diner.members AS m1
RIGHT JOIN dannys_diner.sales AS s
ON m1.customer_id = s.customer_id
LEFT JOIN dannys_diner.menu AS m2
ON m2.product_id = s.product_id
````
![image](https://user-images.githubusercontent.com/104872221/170372976-f0e1f2b7-d268-4b98-89fb-be6fffbd0c33.png)


--------------------------------------------------------------------------------------------------------------------------------------------

````sql
WITH cte_ranking AS (
SELECT
	s.customer_id,
	s.order_date,
	m2.product_name,
	m2.price,
	(CASE WHEN s.order_date >= m1.join_date THEN 'Y'
	ELSE 'N'
	END) AS member
FROM dannys_diner.members AS m1
RIGHT JOIN dannys_diner.sales AS s
ON m1.customer_id = s.customer_id
LEFT JOIN dannys_diner.menu AS m2
ON m2.product_id = s.product_id
)
SELECT 
	*,
	CASE WHEN member = 'N' THEN NULL
	ELSE
		RANK() OVER (PARTITION BY customer_id, member ORDER BY order_date) 
		END AS ranking
FROM cte_ranking
ORDER BY customer_id, order_date;
````

![image](https://user-images.githubusercontent.com/104872221/170373103-5e909c29-eece-4dfa-8112-a518ed47d9a7.png)











