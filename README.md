-- 1 Million Rows Apple dataset project



-- Improving Query Performance

EXPLAIN ANALYZE
SELECT * FROM sales
WHERE product_id = 'P-44'

CREATE INDEX sales_product_id ON sales(product_id);

EXPLAIN ANALYZE
SELECT * FROM sales
WHERE store_id = 'ST-31'

-- et - o.242
-- pt - 0.219

CREATE INDEX sales_store_id ON sales(store_id);


CREATE INDEX sales_sale_date ON sales(sale_date);

-- BUSINESS PROBLEMS 

-- Q.1 - FIND THE NUMBER OF STORES IN EACH COUNTRY.

SELECT country, COUNT(store_id) as total_stores
FROM stores
GROUP BY 1
ORDER BY 1 DESC;

-- Q.2 CALCULATE THE TOTAL NUMBERS OF UNITS SOLD BY EACH STORE.

SELECT  store_id, SUM(quantity) as unit_sold
FROM sales
GROUP BY 1

-- Q.3 IDENTIFY HOW MANY SALES OCCURED IN DEC 2023.

SELECT 
COUNT(sale_id) as total_sale
FROM sales
WHERE TO_CHAR(sale_date, 'MM-YYYY') = '12-2023'

-- Q.4 DETERMINE HOW MANY STORES HAVE NEVER HAD A WARRANTY CLAIM FILED.

SELECT COUNT(*) FROM stores
WHERE store_id NOT IN(
					SELECT DISTINCT store_id
					FROM sales AS s
					RIGHT JOIN warranty AS w
					ON s.sale_id = w.sale_id
					)

-- Q.5 CALCULATE THE PPERCENTAGE OF WARRANTY CLAIM MARKED AS 'REJECTED'.	


SELECT
ROUND(COUNT(sale_id)/(SELECT COUNT(sale_id) FROM warranty) :: numeric
*100, 2)
AS warranty_rejected_percentage
FROM warranty
WHERE repair_status = 'Rejected'


-- Q.6 IDENTIFY WHICH STORE HAS THE HIGHEST TOTAL UNITS SOLD IN THE LAST YEAR.

SELECT s.store_id, st.store_name, COUNT(s.quantity)
FROM sales AS s
JOIN  stores AS st
ON s.store_id = st.store_id
WHERE sale_date >= (CURRENT_DATE - INTERVAL '1 year')
GROUP BY 1,2
ORDER BY COUNT(quantity) DESC
LIMIT 1;  

-- Q.7 COUNT THE NUMBER OF UNIQUE PRODUCTS SOLD IN THE LAST YEAR.

SELECT 
COUNT(DISTINCT product_id)
FROM sales
WHERE sale_date >= (CURRENT_DATE - INTERVAL '1 year')

--Q.8 FIND THE AVG PRICE OF THE PRODUCT IN EACH CATEGORY.

SELECT AVG(p.price) as avg_price, p.category_id, c.category_name
FROM products as p
JOIN category as c
ON p.category_id = c.category_id
GROUP BY 2,3 

-- Q.9 HOW MANY WARRANTY CLAIMS WERE FILED IN 2020?

SELECT claim_id 
FROM warranty
WHERE 
TO_CHAR(claim_date, 'YYYY') = '2024';

--Q.10 F0R EACH STORE, IDENTIFY THE BEST-SELLLING DAY BASED ON HIGHEST QUANTITY SOLD.

SELECT *
FROM
(
SELECT store_id,
		TO_CHAR(sale_date, 'DAY') AS day_name,
		SUM(quantity) AS total_unit_sold,
		RANK() OVER(PARTITION BY store_id ORDER BY SUM(quantity) DESC) as rank 
FROM sales
GROUP BY  1,2
) AS t1
WHERE rank = 1


-- Intermediate Questions

--Q.11 IDENTIFY THE LEAST SELLING PRODUCT IN EACH CATEGORY FOR EACH YEAR BASED ON TOTAL UNITS SOLD.
WITH product_rank
AS
(
SELECT p.product_name, st.country, SUM(s.quantity) AS total_qty_sold,
RANK() OVER(PARTITION BY st.country ORDER BY SUM(s.quantity)) AS rank
FROM 
sales AS s
JOIN stores AS st
ON st.store_id = s.store_id
JOIN products AS p
ON p.product_id = s.product_id
GROUP BY 1,2
)
SELECT * FROM product_rank
WHERE rank = 1

-- Q.12 CALCULATE HOW MANY WARRANTY CLAIMS WERE FILED WITHIN 180 DAYS OF A PRODUCT SALE.


SELECT COUNT(*)
FROM 
warranty AS w
LEFT JOIN
sales AS s
ON s.sale_id = w.sale_id
WHERE 
	w.claim_date - sale_date <= 180

-- Q.13 DETERMINE HOW MANY WARRANTY CLAIMS WERE FILED FOR PRODUCTS LAUNCHED IN THE LAST TWO YEARS.


SELECT 
	p.product_name,
	COUNT(w.claim_id) AS total_claims,
	COUNT(s.sale_id) as total_sales
FROM warranty AS w
RIGHT JOIN sales AS s
ON s.sale_id = w.sale_id
JOIN products AS p
ON p.product_id = s.product_id
WHERE p.launch_date >= CURRENT_DATE - INTERVAL '2 YEARS'
GROUP BY 1


-- Q.14 LIST THE  MONTHS IN THE LAST 3 YEARS WHERE SALES EXCEEDED 5,000 UNITS IN THE USA.

SELECT 
		TO_CHAR(sale_date, 'MM-YYYY') AS sale_month,
		SUM(s.quantity) AS total_unit_sold
FROM
sales AS s
JOIN stores AS st
ON st.store_id = s.store_id
WHERE
st.country = 'United States'
AND s.sale_date >= CURRENT_DATE - INTERVAL '3 YEAR' 
GROUP BY 1
HAVING SUM(s.quantity) > 5000


-- Q.15 IDENTIFY THE PRODUCT CATEGORY WITH THE MOST WARRANTY CLAIMS FILED IN THE LAST TWO YEARS.


SELECT s.product_id, c.category_name, COUNT(w.claim_id) AS total_claims FROM 
warranty AS w
JOIN sales AS s
ON s.sale_id = w.sale_id
JOIN products AS p
ON p.product_id = s.product_id
JOIN category AS c
ON c.category_id = p.category_id
WHERE claim_date >= CURRENT_DATE -INTERVAL '2 YEAR'
GROUP BY 1,2
ORDER BY 3 DESC


-- COMPLEX QUESTIONS 

-- Q.16  DETERMINE THE PERCENTAGE CHANCE OF RECEIVING WARRANTY CALIMS AFTER EACH PURCHASE FOR EACH COUNTRY.


SELECT 
	country,
	total_unit_sold,
	total_claim,
	total_claim::numeric/total_unit_sold :: numeric *100 AS percentage_of_total_claim
FROM 
(SELECT st.country, 
		SUM(s.quantity) AS total_unit_sold,
		COUNT(w.claim_id) AS total_claim
		 FROM stores AS st
JOIN sales AS s
ON s.store_id = st.store_id
JOIN warranty AS w
ON w.sale_id = s.sale_id
GROUP BY 1) t1

-- Q.17 ANALYZE THE YEAR-BY-YEAR GROWTH RATIO FOR EACH STORE.

-- each store and their yearly sale


WITH yearly_sales
AS
(SELECT 
	s.store_id,
	st.store_name,
	EXTRACT(YEAR FROM sale_date) AS year,
	SUM(s.quantity * p.price) AS total_sale
FROM sales AS s
JOIN products AS p
ON s.product_id = p.product_id
JOIN stores AS st
ON s.store_id = st.store_id
GROUP BY 1,2,3
ORDER BY 2,3),

growth_ratio 
AS
(SELECT 
	store_name,
	year,
	LAG(total_sale,1) OVER(PARTITION BY store_name ORDER BY year) as last_year_sale,
	total_sale AS current_year_sale
FROM 
yearly_sales
)
SELECT 
	store_name,
	year,
	last_year_sale,
	current_year_sale,
	ROUND(
			(current_year_sale-last_year_sale):: numeric/
						last_year_sale *100,3)
						AS growth_ratio
FROM
growth_ratio
WHERE 
	last_year_sale IS NOT NULL
	AND
	YEAR <> EXTRACT(YEAR FROM CURRENT_DATE)


-- Q.18 CALCULATE THE CORRELATION BETWEEN PRODUCT PRICE AND WARRANTY CLAIMS FOR PRODUCTS SOLD IN THE LAST FIVE YEARS
-- SEGMENTED BY PRICE RANGE.


SELECT 
		CASE 	
			WHEN p.price <500 THEN 'LESS EXPENSIVE PRODUCT'
			WHEN p.price BETWEEN 500 AND 1000 THEN 'MID RANGE PRODUCT'
			ELSE 'EXPENSIVE PRODUCT'
		END AS price_segment,
		COUNT(w.claim_id) AS total_claim
FROM warranty AS w
LEFT JOIN 
sales AS s
ON w.sale_id = s.sale_id
JOIN products AS p
ON p.product_id = s.product_id
WHERE claim_date >= CURRENT_DATE - INTERVAL '5 YEAR'
GROUP BY 1

-- Q.19 IDENTIFY THE STORE WITH THE HIGHEST PERCENTAGE OF 'COMPLETELY REPAIRED' CLAIMS RELATIVE TO TOTAL CLAIMS FILED.

WITH completed_repaired
AS
(SELECT 
	s.store_id,
	COUNT(w.claim_id)
FROM sales AS s
JOIN warranty AS w
ON w.sale_id = s.sale_id
WHERE w.repair_status = 'Completed'
GROUP BY 1),

total_repaired
AS (SELECT 
	s.store_id,
	COUNT(w.claim_id)
FROM sales AS s
JOIN warranty AS w
ON w.sale_id = s.sale_id
GROUP BY 1)

SELECT 
	tr.store_id,
	cr.completed_repaired,
	tr.total_repaired,
	ROUND(cr.completed_repaired::numeric/tr.total_repaired::numeric*100,2) AS percentage_repaired
FROM Completed_repaired as cr
JOIN total_repaired as tr
On tr.store_id = cr.store_id



-- Q.20 WRITE A QUERY TO CALCULCATE THE MONTHLY RUNNING TOTAL OF SALES FOR EACH STORE
-- OVER THE PAST FOUR YEARS AND COMPARE TRENDS DURING THIS PERIOD.

WITH monthly_sale
AS
(SELECT 
	store_id,
	EXTRACT (YEAR FROM sale_date) AS year,
	EXTRACT ( MONTH FROM sale_date) AS month,
	SUM(p.price * s.quantity) total_revenue
FROM sales AS s
JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2,3
ORDER BY 1,2,3
)
SELECT 
		store_id,
		month,
		year,
		total_revenue,
		SUM(total_revenue) OVER (PARTITION BY store_id ORDER BY year,month) as running_total
FROM monthly_sale


-- Q.BONUS QUESTION --> ANALYZE THE PRODUCT SALES, TRENDS OVER TIME, SEGMENT INTO KEY PERIODS: FROM LAUNCH TO 6  MONTHS
-- 6-12 MONTHS, 12-18 MONTHS AND BEYOND 18 MONTHS.

SELECT 
	p.product_name,
	CASE 
		WHEN s.sale_date BETWEEN p.launch_date AND p.launch_date + INTERVAL '6 month' THEN '0-6 MONTH'
		WHEN s.sale_date BETWEEN  p.launch_date + INTERVAL '6 MONTH' AND p.launch_date + INTERVAL '12 MONTH' THEN '6-12'
		WHEN s.sale_date BETWEEN  p.launch_date + INTERVAL '12 MONTH' AND p.launch_date + INTERVAL '18 MONTH' THEN '6-12'
		ELSE '18+'
	END AS product_sale
	SUM(s.quantity) AS total_qty_sold
FROM sales AS s
JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2
ORDER BY 1,3 DESC









