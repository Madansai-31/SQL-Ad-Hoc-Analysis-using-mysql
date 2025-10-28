# Ad-Hoc-Analysis-using-mysql
SQL ad-hoc analysis for AtliQ Hardwares (consumer goods). MySQL queries, results, screenshots, sample data and visuals solving 10 business questions.
---

<span style="color:#9ca3af">**--> 1. Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region**</span> 
```sql
SELECT	distinct(market) 
FROM dim_customer
WHERE customer = "Atliq Exclusive"  && region = "APAC"
```
<span style="color:#9ca3af">-->2. What is the percentage of unique product increase in 2021 vs. 2020? The final output contains these fields, unique_products_2020 unique_products_2021 percentage_chg</span>
```sql
WITH unique_20 AS(
SELECT count(DISTINCT(product_code)) AS unique_product_2020,fiscal_year
FROM fact_sales_monthly
WHERE fiscal_year = 2020),
unique_21 AS(
SELECT count(DISTINCT(product_code)) AS unique_product_2021,fiscal_year
FROM fact_sales_monthly
WHERE fiscal_year = 2021)
SELECT unique_21.unique_product_2021,unique_20.unique_product_2020, 
(unique_21.unique_product_2021-unique_20.unique_product_2020)*100/unique_20.unique_product_2020 AS pct_chg
FROM unique_20
CROSS JOIN unique_21
```
<span style="color:#9ca3af">-->3. Provide a report with all the unique product counts for each segment and sort them in descending order of product counts. The final output contains 2 fields, segment product_count</span>
```sql
SELECT segment ,count(distinct(product)) AS pd_count 
FROM dim_product
GROUP BY segment
ORDER BY pd_count DESC
```
<span style="color:#9ca3af">-->4. Follow-up: Which segment had the most increase in unique products in 2021 vs 2020? The final output contains these fields, segment product_count_2020 product_count_2021 difference</span>
```sql
WITH yearly_counts AS (
    SELECT 
        COUNT(DISTINCT dp.product) AS product_count,
        segment, fiscal_year
    FROM 
        fact_sales_monthly as f
	join
		dim_product as dp
	on
		f.product_code = dp.product_code
    WHERE 
        fiscal_year IN (2020, 2021)
    GROUP BY 
        segment, fiscal_year
)
SELECT
    segment,
    SUM(CASE WHEN fiscal_year = 2021 THEN product_count END) AS unique_product_2021,
    SUM(CASE WHEN fiscal_year = 2020 THEN product_count END) AS unique_product_2020,
    (SUM(CASE WHEN fiscal_year = 2021 THEN product_count END) - SUM(CASE WHEN fiscal_year = 2020 THEN product_count END)) as diff
FROM
    yearly_counts
GROUP BY segment
```
<span style="color:#9ca3af">-->5. Get the products that have the highest and lowest manufacturing costs. The final output should contain these fields, product_code product manufacturing_cost</span>
```sql
WITH cost_summary AS (
SELECT 
	dp.product,
	dp.product_code,
	f.manufacturing_cost
FROM fact_manufacturing_cost f
JOIN dim_product dp
	ON dp.product_code = f.product_code
),
agg_cost AS (
    SELECT 
        product,
        product_code,
        MAX(manufacturing_cost) AS max_cost,
        MIN(manufacturing_cost) AS min_cost
    FROM cost_summary    
    GROUP BY product, product_code
),
highest AS (
    SELECT *
    FROM agg_cost
    ORDER BY max_cost DESC
    LIMIT 1
),
lowest AS (
    SELECT *
    FROM agg_cost
    ORDER BY min_cost ASC
    LIMIT 1
)
SELECT * FROM highest
UNION ALL
SELECT * FROM lowest
```
<span style="color:#9ca3af">-->6. Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market. The final output contains these fields, customer_code customer average_discount_percentage</span>
```sql
SELECT dc.customer_code,
dc.customer,
ROUND(avg(f.pre_invoice_discount_pct),2) as dis_pct
FROM fact_pre_invoice_deductions f
JOIN dim_customer dc
	ON f.customer_code = dc.customer_code
WHERE f.fiscal_year = 2021 && dc.market = "India"
GROUP BY dc.customer_code,dc.customer
ORDER BY dis_pct DESC
LIMIT 5
```
<span style="color:#9ca3af">-->7. Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month . This analysis helps to get an idea of low and high-performing months and take strategic decisions. The final report contains these columns: Month Year Gross sales Amount</span>
```sql
with sales_data as 
(SELECT  fs.product_code,
fs.customer_code,
fs.date,
fs.sold_quantity*fg.gross_price as gross_sales
FROM fact_sales_monthly fs
JOIN fact_gross_price fg
	ON fs.product_code = fg.product_code AND fs.fiscal_year = fg.fiscal_year
JOIN dim_customer dc
	ON fs.customer_code = dc.customer_code
WHERE customer = "Atliq Exclusive" )
SELECT year(date) as year, 
monthname(date) as month,
round((sum(gross_sales))/100000,2) as gross_sales_in_thousands
FROM sales_data
GROUP BY year,month
```
<span style="color:#9ca3af">-->8. In which quarter of 2020, got the maximum total_sold_quantity? The final output contains these fields sorted by the total_sold_quantity, Quarter total_sold_quantity</span>
```sql
SELECT 
	CASE
		WHEN MONTH(date) IN (9,10,11) THEN "Q1"
		WHEN MONTH(date) IN (12,1,2) THEN "Q2"
		WHEN MONTH(date) IN (3,4,5) THEN "Q3"
		ELSE "Q4"
	END AS quarter,sum(sold_quantity)/100000 as total_quantity_in_thousands
FROM fact_sales_monthly
WHERE fiscal_year = 2020
GROUP BY quarter
```
<span style="color:#9ca3af">-->9. Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution? The final output contains these fields, channel gross_sales_mln percentage</span>
```sql
WITH sales_data as(
SELECT dc.channel,
round(sum(f.sold_quantity*fg.gross_price)/1000000,2) as gross_sales_mln
FROM fact_sales_monthly f
JOIN dim_customer dc
	ON f.customer_code = dc.customer_code
JOIN fact_gross_price fg
	ON fg.product_code = f.product_code AND fg.fiscal_year = f.fiscal_year
WHERE f.fiscal_year = 2021
GROUP BY dc.channel)
SELECT channel,gross_sales_mln,
round(gross_sales_mln*100/sum(gross_sales_mln)over(),2) as pct
FROM sales_data
ORDER BY pct desc
```
<span style="color:#9ca3af">-->10. Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021? The final output contains these fields, division product_code product total_sold_quantity rank_order</span>
```sql
WITH product_sales as(
SELECT dp.division,
dp.product_code,
dp.product,
sum(f.sold_quantity) as total_sold_quantity,
RANK() OVER(PARTITION BY division ORDER BY sum(f.sold_quantity) DESC) as rnk
FROM fact_sales_monthly f
JOIN dim_product dp
ON dp.product_code = f.product_code
GROUP BY dp.product,dp.product_code,dp.division)
SELECT division,product_code,product,total_sold_quantity,rnk
FROM product_sales
WHERE rnk<=3
ORDER BY division,rnk DESC
```
