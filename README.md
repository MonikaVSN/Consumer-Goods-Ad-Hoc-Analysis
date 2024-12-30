# Consumer-Goods-Ad-Hoc-Analysis
**Project Description**
AtliQ Hardware is one of the leading computer hardware producers in India and well expanded in other countries too.AtliQ sells products in different segments like
Peripherals and Accessories,PC and Network and Storage in Platforms like Brick & Mortar (Chroma , Best buy) and E-Commerce (Amazon, Flipkart)
## Ad-Hoc-Requests
**MYSQl Codes**
**1.Provide list of markets in which customer "AtliQ Exclusive"operates its business in APAC region**
      
      select distinct market,COUNT(CUSTOMER)
      
      from dim_customer
      
      where customer='Atliq Exclusive' and region='APAC'
  
**2.What is the percentage of unique product increase in 2021 vs. 2020?**
      
      with cte as
      
      (select count(distinct product_code) as unique_products_2020,
      		
        (select count(distinct product_code)
      		
        from fact_sales_monthly
      		
        where fiscal_year=2021) as unique_products_2021
      
      from fact_sales_monthly
      
      
      where fiscal_year=2020)
      
      select  *,
      	
       round((unique_products_2021-unique_products_2020)*100/unique_products_2020,2) as percentage_change
      
      from cte

**3.Provide a report with all the unique product counts for each  segment  and sort them in descending order of product counts.**
    select  segment,
    		count(distinct product_code) as product_count
    from dim_product
    group by segment 
    order by product_count desc

**4.Follow-up: Which segment had the most increase in unique products in 2021 vs 2020?**
    with cte1 as 
    (select segment,count(distinct p.product_code) as product_count_2020
    from dim_product p
    join fact_sales_monthly s using(product_code)
    where fiscal_year=2020
    group by segment 
    order by product_count_2020 desc),
    cte2 as (select segment,count(distinct p.product_code) as product_count_2021
    from dim_product p
    join fact_sales_monthly s using(product_code)
    where fiscal_year=2021
    group by segment 
    order by product_count_2021 desc)
    select 
    	cte1.segment, cte2.product_count_2021,
    	cte1.product_count_2020,
    	(cte2.product_count_2021)-(cte1.product_count_2020) as difference
    from cte1
    join cte2 using(segment)
    order by difference desc

**5.Get the products that have the highest and lowest manufacturing costs.The final output should contain these fields, product_code product manufacturing_cost**
    select  p.product_code, 
    		product, 
    		m.manufacturing_cost
    from dim_product p
    join fact_manufacturing_cost m using(product_code)
    where 
    	m.manufacturing_cost = (select max(manufacturing_cost) from fact_manufacturing_cost) or
    	m.manufacturing_cost = (select min(manufacturing_cost) from fact_manufacturing_cost)
    order by manufacturing_cost desc

**6.Generate a report which contains the top 5 customers who received an average high  pre_invoice_discount_pct for fiscal  year 2021  and in the Indian  market.**
    select  c.customer_code, 
    		customer, 
    		round(avg(pre_invoice_discount_pct)*100,2) as average_discount_percentage
    from fact_pre_invoice_deductions pre
    join dim_customer c using(customer_code)
    where fiscal_year = "2021" and market = "India"
    group by c.customer_code 
    order by average_discount_percentage desc
    limit 5

**7.Get the complete report of the Gross sales amount for the customer  “Atliq Exclusive” for each month.This analysis helps to  get an idea of low and high-performing months and take strategic decisions.**
    select  f.fiscal_year as fiscal_year,
    		date_format(date, '%M(%Y)') as month,
    		concat(round(sum(sold_quantity * g.gross_price)/1000000,2)," M") as gross_sales
    from fact_sales_monthly f
    join fact_gross_price g using(product_code)
    join dim_customer c using(customer_code)
    where customer= "Atliq Exclusive" and f.fiscal_year in (2020,2021)
    group by month , fiscal_year

**8.In which quarter of 2020, got the maximum total_sold_quantity? The final output contains these fields sorted by the total_sold_quantity**
    SELECT  
        CASE
            WHEN MONTH(date) IN (9 , 10, 11) THEN 'Q1'
            WHEN MONTH(date) IN (12 , 1, 2) THEN 'Q2'
            WHEN MONTH(date) IN (3 , 4, 5) THEN 'Q3'
            WHEN MONTH(date) IN (6 , 7, 8) THEN 'Q4'
        END AS quarter,
        concat(round(sum(sold_quantity)/1000000,2), " M") as total_sold_quantity
    FROM fact_sales_monthly
    where fiscal_year = '2020'
    GROUP BY quarter

**9.Which channel helped to bring more gross sales in fiscal year 2021 and the percentage of contribution?**
    with cte as 
    (select channel,
    		concat(round(sum(sold_quantity * g.gross_price)/1000000,2)," M") as gross_sales
    from fact_sales_monthly s
    join fact_gross_price g 
    on s.product_code = g.product_code and s.fiscal_year = g.fiscal_year
    join dim_customer c using(customer_code)
    where s.fiscal_year='2021'
    group by channel
    order by gross_sales desc)
    select  *,
    		round((gross_sales/sum(gross_sales) over()*100),2)  as percentage
    from cte

**10.Get Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021?**
    with cte as 
    (select division,
            p.product_code,
            product,
    		sum(sold_quantity) as total_sold_quantity
    from gdb0041.fact_sales_monthly s
    join dim_product p using(product_code)
    where fiscal_year = '2021'
    group by division,p.product_code, product),
    rnk as (select  *,
    		dense_rank() over(partition by division order by total_sold_quantity desc) as rank_order
    from cte
    order by total_sold_quantity desc)
    select * from rnk 
    where rank_order <=3
