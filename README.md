# 🍜 Case Study #1: Danny's Diner 
<img src="https://user-images.githubusercontent.com/81607668/127727503-9d9e7a25-93cb-4f95-8bd0-20b87cb4b459.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Dataset](#dataset)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent, and also which menu items are their favorite. 

***

## Dataset

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***

## Question and Solution

**1. What is the total amount each customer spent at the restaurant?**
- Join `sales` and `menu` tables to get the `price` for each `product_id`d sold in `sales` 
- Sum the total sales contributed by each customer. CONCAT() adds the $ sign
- Group the aggregated results by `customer_id`.
  
````sql
SELECT s.customer_id, 
       CONCAT('$ ', SUM(m.price)) AS total_sales
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
  ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
````

#### Answer:
| customer_id | total_sales |
| ----------- | ----------- |
| A           | $ 76        |
| B           | $ 74        |
| C           | $ 36        |


***

**2. How many days has each customer visited the restaurant?**
  
````sql
SELECT s.customer_id, 
       COUNT(DISTINCT s.order_date) AS visit_count
FROM dannys_diner.sales AS s
GROUP BY customer_id;
````

#### Answer:
| customer_id | visit_count |
| ----------- | ----------- |
| A           | 4           |
| B           | 6           |
| C           | 2           |

***

**3. What was the first item from the menu purchased by each customer?**
- Create a Common Table Expression (CTE) named `ranked` used to rank the orders based on `customer_id` and `order_date`. There is no timestamp, so based on the date customer A has both sushi and curry as the first order on that day. 
- In the outer query, filtered results in the **WHERE** clause for rank = 1 which represent the first order(s) placed 
- Use the GROUP BY clause to group the result by `customer_id`, `product_name`.

````sql
WITH ranked AS (
            SELECT s.*,
                   m.product_name,
                   DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rank
            FROM dannys_diner.sales AS s
            JOIN dannys_diner.menu AS m
              ON s.product_id = m.product_id
)

SELECT customer_id, 
       product_name AS first_product_ordered,
       TO_CHAR(order_date, 'YYYY-MM-DD') AS date_ordered
FROM ranked
WHERE rank = 1
GROUP BY customer_id, first_product_ordered, date_ordered;
````

#### Answer:
| customer_id | first_product_ordered | date_ordered |
| ----------- | --------------------- | ------------ |
| A           | curry                 | 2021-01-01   |
| A           | sushi                 | 2021-01-01   |
| B           | curry                 | 2021-01-01   |
| C           | ramen                 | 2021-01-01   |

***

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**

````sql
SELECT m.product_name,
       COUNT(s.product_id) AS order_count
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
  ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY order_count DESC
LIMIT 1;
````

#### Answer:
| product_name | order_count | 
| ------------ | ----------- |
| ramen        | 8           |

***

**5. Which item was the most popular for each customer?**
- Create a Common Table Expression (CTE) named `order_info` by joining the `menu` and `sales` tables using the `product_id` column.
- Group the results by `sales.customer_id` and `menu.product_name`, calculating the count of occurrences for each group using `COUNT(menu.product_id)`.
- Use the **DENSE_RANK** window function to determine the ranking of each `sales.customer_id` partition based on the order count, sorted in descending order.
- In the outer query, apply **WHERE** filter to retrieve rank = 1, representing the highest order count for each customer.
  
````sql
WITH order_info AS (
  		SELECT product_name,
                       customer_id,
                       COUNT(product_name) AS order_count,
		       DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY COUNT(product_name) DESC) AS rank_num
   		FROM dannys_diner.menu
   		JOIN dannys_diner.sales 
 		  ON menu.product_id = sales.product_id
 		GROUP BY customer_id,
            		 product_name
)
  
SELECT customer_id,
       product_name,
       order_count
FROM order_info
WHERE rank_num =1;
````

#### Answer:
| customer_id | product_name | order_count |
| ----------- | ------------ |------------ |
| A           | ramen        |  3          |
| B           | ramen        |  2          |
| B           | curry        |  2          |
| B           | sushi        |  2          |
| C           | ramen        |  3          |


***

**6. Which item was purchased first by the customer after they became a member?**
- Common Table Expression (CTE) named `joined_as_memeber` performs ranking of what has been ordered first
- Outer query is used to select rank = 1 and get the name of dishes
  
```sql
WITH joined_as_member AS (
 		        SELECT m.*, 
			       s.order_date,
        		       s.product_id,
			       ROW_NUMBER() OVER (PARTITION BY m.customer_id ORDER BY s.order_date) AS rank
		    	FROM dannys_diner.members m
			JOIN dannys_diner.sales s
			  ON m.customer_id = s.customer_id
			 AND s.order_date > m.join_date
)

SELECT customer_id, 
       product_name 
FROM joined_as_member
JOIN dannys_diner.menu
  ON joined_as_member.product_id = menu.product_id
WHERE rank = 1
ORDER BY customer_id;
```

#### Answer:
| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |

***

**7. Which item was purchased just before the customer became a member?**

````sql
WITH joined_as_member AS (
 		        SELECT m.*, 
			       s.order_date,
        		       s.product_id,
			       ROW_NUMBER() OVER (PARTITION BY m.customer_id ORDER BY s.order_date DESC) AS rank
		    	FROM dannys_diner.members m
			JOIN dannys_diner.sales s
			  ON m.customer_id = s.customer_id
			 AND s.order_date < m.join_date
)

SELECT customer_id, 
       product_name 
FROM joined_as_member
JOIN dannys_diner.menu
  ON joined_as_member.product_id = menu.product_id
WHERE rank = 1
ORDER BY customer_id;
````

#### Answer:
| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | sushi        |

***

**8. What is the total items and amount spent for each member before they became a member?**
- Count `sales.product_id` as total_items for each customer and the sum of `menu.price` as total_sales IN $.
- From `dannys_diner.sales` table, join `dannys_diner.members` table on `customer_id` column, ensuring that `sales.order_date` is earlier than `members.join_date` (`sales.order_date < members.join_date`).
- Then, join `dannys_diner.menu` table to `dannys_diner.sales` table on `product_id` column.
- Group the results by `sales.customer_id`.
  
```sql
SELECT s.customer_id,
       COUNT(m.product_name) AS total_items,
       CONCAT('$ ', SUM(m.price)) AS amount_spent
FROM dannys_diner.menu AS m
JOIN dannys_diner.sales AS s 
  ON m.product_id = s.product_id
JOIN dannys_diner.members AS mem 
  ON mem.customer_id = s.customer_id
WHERE s.order_date < mem.join_date
GROUP BY s.customer_id
ORDER BY customer_id;
```

#### Answer:
| customer_id | total_items | amount_spent |
| ----------- | ----------- | ------------ |
| A           | 2           | $ 25         |
| B           | 3           | $ 40         |

***

**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?**

```sql
SELECT s.customer_id,
       SUM(CASE
               WHEN product_name = 'sushi' THEN price * 20
               ELSE price * 10
           END) AS total_points
FROM dannys_diner.menu AS m
JOIN dannys_diner.sales AS s
  ON m.product_id = s.product_id
GROUP BY customer_id
ORDER BY customer_id;
```

#### Answer:
| customer_id | total_points | 
| ----------- | ------------ |
| A           | 860          |
| B           | 940          |
| C           | 360          |

***

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?**

#### Assumptions:
- During the initial period (Day -X to Day 1), members earn 10 points for every $1 spent, and for sushi, it's 20 points per $1.
- In the first week (Day 1 to Day 7) of membership, all items earn 20 points per $1 spent.
- From Day 8 to the end of January 2021, members earn 10 points for every $1 spent, with sushi earning double at 20 points per $1."

#### Steps:
- Create a CTE named `dates_cte`.
- Calculate `valid_date` in `dates_cte` by adding 6 days to `join_date` and determine the month's `last_date` by subtracting 1 day from January 2021's last day.
- Join `dannys_diner.sales` with `dates_cte` on `customer_id`, ensuring `order_date` is after `join_date` and not later than `last_date`.
- Join with `dannys_diner.menu` based on `product_id`.
- In the outer query, calculate points using a **CASE** statement. Multiply sushi prices by 20 points (2 * 10) during specified periods; for other products, multiply prices by 10 points.
- Calculate the sum of points for each customer.
  
```sql
WITH dates_cte AS (
                SELECT customer_id, 
                       join_date, 
                       join_date + 6 AS valid_date, 
                       DATE_TRUNC('month', '2021-01-31'::DATE)
        	                   + interval '1 month' 
                                   - interval '1 day' AS last_date
                FROM dannys_diner.members
)

SELECT s.customer_id, 
       SUM(CASE
           WHEN m.product_name = 'sushi' THEN 2 * 10 * m.price
           WHEN s.order_date BETWEEN d.join_date AND d.valid_date THEN 2 * 10 * m.price
           ELSE 10 * m.price END) AS total_points
FROM dannys_diner.sales AS s
JOIN dates_cte AS d
  ON s.customer_id = d.customer_id
 AND d.join_date <= s.order_date
 AND s.order_date <= d.last_date
JOIN dannys_diner.menu AS m
  ON s.product_id = m.product_id
GROUP BY s.customer_id;
```

#### Answer:
| customer_id | total_points | 
| ----------- | ------------ |
| A           | 1020         |
| B           | 320          |


***

## BONUS QUESTION

**Recreate the table with: customer_id, order_date, product_name, price, member (Y/N). Danny also requires further information about the ```ranking``` of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ```ranking``` values for the records when customers are not yet part of the loyalty program.**

```sql
WITH cte AS (
           SELECT s.*, 
    		  menu.*,
 		  CASE
		     WHEN m.join_date > s.order_date THEN 'N'
		     WHEN m.join_date <= s.order_date THEN 'Y'
		     ELSE 'N' END AS member_status
	   FROM dannys_diner.sales AS s
  	   LEFT JOIN dannys_diner.members AS m
    		  ON s.customer_id = m.customer_id
	   JOIN dannys_diner.menu
	     ON s.product_id = menu.product_id
)

SELECT cte.customer_id, cte.order_date, cte.product_name, cte.price, cte.member_status, 
       CASE
    	 WHEN member_status = 'N' then NULL
    	 ELSE DENSE_RANK () OVER (PARTITION BY customer_id, member_status ORDER BY order_date) END AS ranking
FROM cte;
```

#### Answer: 
| customer_id | order_date | product_name | price | member_status | ranking | 
| ----------- | ---------- | -------------| ----- | ------------- |-------- |
| A           | 2021-01-01 | sushi        | 10    | N             | NULL    |
| A           | 2021-01-01 | curry        | 15    | N             | NULL    |
| A           | 2021-01-07 | curry        | 15    | Y             | 1       |
| A           | 2021-01-10 | ramen        | 12    | Y             | 2       |
| A           | 2021-01-11 | ramen        | 12    | Y             | 3       |
| A           | 2021-01-11 | ramen        | 12    | Y             | 3       |
| B           | 2021-01-01 | curry        | 15    | N             | NULL    |
| B           | 2021-01-02 | curry        | 15    | N             | NULL    |
| B           | 2021-01-04 | sushi        | 10    | N             | NULL    |
| B           | 2021-01-11 | sushi        | 10    | Y             | 1       |
| B           | 2021-01-16 | ramen        | 12    | Y             | 2       |
| B           | 2021-02-01 | ramen        | 12    | Y             | 3       |
| C           | 2021-01-01 | ramen        | 12    | N             | NULL    |
| C           | 2021-01-01 | ramen        | 12    | N             | NULL    |
| C           | 2021-01-07 | ramen        | 12    | N             | NULL    |

***
