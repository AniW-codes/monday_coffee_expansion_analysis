# Monday Coffee Expansion SQL Project

**Project Category: Advanced**

**Dataset**: [Monday Coffee Expansion Dataset](https://www.kaggle.com/datasets/najir0123/monday-coffee-sql-data-analysis-project/)

**Tableau Analysis** : [Tableau Dashboard]()

![Company Logo](https://github.com/najirh/Monday-Coffee-Expansion-Project-P8/blob/main/1.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.



```sql
-- Monday Coffee SCHEMAS

DROP TABLE IF EXISTS sales;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS city;

-- Import Rules
-- 1st import to city
-- 2nd import to products
-- 3rd import to customers
-- 4th import to sales


CREATE TABLE city
(
	city_id	INT PRIMARY KEY,
	city_name VARCHAR(15),	
	population	BIGINT,
	estimated_rent	FLOAT,
	city_rank INT
);

CREATE TABLE customers
(
	customer_id INT PRIMARY KEY,	
	customer_name VARCHAR(25),	
	city_id INT,
	CONSTRAINT fk_city FOREIGN KEY (city_id) REFERENCES city(city_id)
);


CREATE TABLE products
(
	product_id	INT PRIMARY KEY,
	product_name VARCHAR(35),	
	Price float
);


CREATE TABLE sales
(
	sale_id	INT PRIMARY KEY,
	sale_date	date,
	product_id	INT,
	customer_id	INT,
	total FLOAT,
	rating INT,
	CONSTRAINT fk_products FOREIGN KEY (product_id) REFERENCES products(product_id),
	CONSTRAINT fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id) 
);

-- END of SCHEMAS

select * from city
select * from products
select * from customers
select * from sales


```


## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?
```sql

Select city_name,
		population,
		ROUND(population/1000000,2) as popluation_in_millions,
		ROUND(population*0.25/1000000,2) as estimated_consumers_in_millions
from city


```


2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?

```sql


select city_name,
		SUM(total) as total_revenue
from sales
join customers
		on sales.customer_id = customers.customer_id
join city 
		on customers.city_id = city.city_id
where 	Extract(Year from sale_date) = 2023
		and 
		Extract(Quarter from sale_date) = 4
group by city_name
order by 2 desc




```

3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?

```sql

select product_name,
		Count(*) as total_items_sold,
		SUM(total) as total_revenue_generated
from products
left join sales
	on sales.product_id = products.product_id
group by product_name
order by 2 desc



```

4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
```sql

select 	city_name, 
		sum(total) as total_sale_per_customer,
		Count(Distinct(customers.customer_id)) as total_customers,
		ROUND(
			sum(total)::numeric/Count(Distinct(customers.customer_id))::numeric
			,2)	as avg_sales_per_customer
from customers
left join sales
	on sales.customer_id = customers.customer_id
left join city
	on customers.city_id = city.city_id
Group by city_name
order by avg_sales_per_customer desc



```

5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers (25%). [Join two CTEs]
```sql


With CTE_city_table as(
					select city_name,
							ROUND(population*0.25/1000000,2) as estimated_consumers_in_millions
					from city
				),
CTE_customers_table
AS
				(
					Select city_name,
							Count(Distinct(customers.customer_id)) as distinct_customers
						from sales
							join customers
									on sales.customer_id = customers.customer_id
							join city 
									on customers.city_id = city.city_id
							group by city_name
							order by 2 desc
				)

Select CTE_city_table.city_name, CTE_customers_table.distinct_customers, CTE_city_table.estimated_consumers_in_millions
from CTE_city_table
join CTE_customers_table
	on CTE_city_table.city_name = CTE_customers_table.city_name
order by 2 desc




```

6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?
```sql


With CTE_Volume as(
			Select city_name,
					product_name,
					Count(sale_id) as total_orders,
					Dense_Rank() Over(Partition by city_name order by Count(sale_id) desc) as Rank_of_orders
			from sales
			join products
				on products.product_id = sales.product_id
			join customers
				on customers.customer_id = sales.customer_id
			join city
				on city.city_id = customers.city_id
			group by 1,2
			order by 1,3 desc
)

Select * 
from CTE_Volume
	where Rank_of_orders <=3




```

7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products? (Coffee related products are ID'ed from 1 to 14)
```sql

Select city_name,
		Count(Distinct(customers.customer_id))
from products
			join sales
				on products.product_id = sales.product_id
			join customers
				on customers.customer_id = sales.customer_id
			join city
				on city.city_id = customers.city_id
where products.product_id <= 14
group by 1
--order by 2 desc



```

8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer. (avg sale v rent) [Join two CTEs]
```sql


With CTE_customers 
as
(
		select 	city.city_name, 
				Count(Distinct(customers.customer_id)) as total_customers,
				ROUND(
					sum(total)::numeric/Count(Distinct(customers.customer_id))::numeric
					,2)	as avg_sales_per_customer
		from customers
		left join sales
			on sales.customer_id = customers.customer_id
		left join city
			on customers.city_id = city.city_id
		Group by 1
		order by 2 desc
),
CTE_Rent
as
			(Select city_name,
					estimated_rent
				from city
			)

Select 	CTE_Rent.city_name, 
		CTE_Rent.estimated_rent,
		CTE_customers.total_customers,
		CTE_customers.avg_sales_per_customer,
		CTE_Rent.estimated_rent/CTE_customers.total_customers as avg_rent_per_customer
from CTE_customers
join CTE_Rent
	on CTE_customers.city_name = CTE_Rent.city_name
order by 4 desc




```

9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly) by each city. [Join two CTEs and using LAG() windows fn]
```sql


With CTE_monthly_sale
as
(
select 
		city_name,
		Extract(Month from sale_date) as month,
		Extract(Year from sale_date) as year,
		Sum(total) as total_sale
from sales
		left join customers
			on sales.customer_id = customers.customer_id
		left join city
			on customers.city_id = city.city_id
group by 1,2,3
order by 1,3,2
),
CTE_growth_ratio
as
(					Select
						city_name,
						month,
						year,
						total_sale as current_month_sale,
						LAG(total_sale, 1) Over(Partition by city_name order by year, month) as previous_month_sale
				from CTE_monthly_sale
)

Select
	city_name,
	month,
	year,
	current_month_sale,
	previous_month_sale,
	ROUND((current_month_sale - previous_month_sale)::numeric/previous_month_sale::numeric * 100,2) as growth_percent
from CTE_growth_ratio
	where previous_month_sale is not null




```

10. **Market Potential Analysis**  
Identify top 3 city based on highest sales, return city name, total sale, average rent per customer, average sales per customer, total customers, estimated coffee consumer (25%) [Join two CTEs]

```sql


With CTE_customers 
as
(
		select 	city.city_name, 
				Count(Distinct(customers.customer_id)) as total_customers,
				SUM(total) as total_sales_revenue,
				ROUND(
					sum(total)::numeric/Count(Distinct(customers.customer_id))::numeric
					,2)	as avg_sales_per_customer
		from customers
		left join sales
			on sales.customer_id = customers.customer_id
		left join city
			on customers.city_id = city.city_id
		Group by 1
		order by 2 desc
),
CTE_Rent
as
			(Select city_name,
					estimated_rent,
					population,
					ROUND((population * 0.25/1000000),2) as estimated_coffee_consumers_in_millions
				from city
			)

Select 	CTE_Rent.city_name, 
		CTE_customers.total_sales_revenue,
		CTE_Rent.estimated_rent,
		CTE_customers.total_customers,
		CTE_Rent.estimated_coffee_consumers_in_millions,
		CTE_customers.avg_sales_per_customer,
		ROUND(CTE_Rent.estimated_rent::numeric/CTE_customers.total_customers::numeric,2) as avg_rent_per_customer
from CTE_customers
join CTE_Rent
	on CTE_customers.city_name = CTE_Rent.city_name
order by 2 desc

```
    

## Recommendations
After analyzing the data, the recommended top three cities for new store openings/expansions are:

**City 1: Pune**  
1.Average rent per customer is relatively low at 294.23.

2.Highest total revenue or sales at 1.2 million.

3.Average sales per customer highest at 24.1k.

**City 2: Delhi**  
1.Highest estimated coffee consumers at 7.7 million.

2.Total number of customers one of the hightst, 68.

3.Average rent per customer is 330.88 (still under 500).

**City 3: Jaipur**  
1.Highest number of customers, 69.

2.Average rent per customer is very low at 156.52.

3.Average sales per customer is relatively good at 11.6k.

---
