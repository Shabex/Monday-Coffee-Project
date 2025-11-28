# Monday Coffee Expansion SQL Project

![Company Logo](https://github.com/najirh/Monday-Coffee-Expansion-Project-P8/blob/main/1.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee. This company has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?
```sql
select 
		city_name,
		round((population * 0.25)/1000000,2) as Total_consumers_millions,
		city_rank
	from city 
	order by Total_consumers_millions desc;
```

2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
```sql
select 
		case
				when extract(quarter from sale_date) = 1 then 'Q1'
				when extract(quarter from sale_date) = 2 then 'Q2'
				when extract(month from sale_date) between 7 and 9 then 'Q3'
				when extract(month from sale_date) between 10 and 12 then 'Q4'
			end As quaters
	from sales;

		
with t1 as 
		(
			select 
					ci.city_name,
					extract(year from sl.sale_date) as year,
					extract(quarter from sl.sale_date) as quater,
					sum(sl.total)as total_sales
				from sales as sl
					join customers as cu
						on sl.customer_id = cu.customer_id
					join city as ci
						on ci.city_id = cu.city_id
				where extract(quarter from sale_date) = 4
					and 
						extract(year from sale_date) = 2023
				group by ci.city_name,year, quater
				order by total_sales desc
			)
	select 
			sum(total_sales)
		from t1;	

```

3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?
```sql
select * from products;
select * from sales;

select 
		pr.product_name,
		count(sl.product_id)
	from products as pr
		left join sales as sl
			on pr. product_id = sl.product_id
	group by pr.product_name
	order by 2 desc;
```

4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
```sql
select * from city;
select * from customers;
select * from sales;

select 
		ci.city_name,
		sum(sl.total) as total_sale,
		count(distinct cu.customer_id) as total_customer,
		sum(sl.total)/count(distinct cu.customer_name) as avg_sale
	from sales as sl
		join customers as cu
			on cu.customer_id = sl.customer_id
		join city as ci
			on ci.city_id = cu.city_id
	group by ci.city_name 
	order by total_sale desc;

```

5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers.
```sql
with t1 as 
		(
			select 
					city_name,
					population*.25 as total_consumers
				from city
		),
		t2 as
		(
			select 
					city_name,
					count(distinct cu.customer_id) as unique_consumer
				from  sales as sl
					join customers as cu
						on sl.customer_id = cu.customer_id
					join city as ci
						on cu.city_id = ci.city_id
				group by ci.city_name
		)
	select 
			t1.city_name,
			t1.total_consumers,
			t2.unique_consumer
		from t1
			join t2
				on t1.city_name = t2.city_name;
			
```

6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?
```sql
with t1 as 
		(
			select 
					pr.product_name,
					ci.city_name,
					count(sl.sale_id) as no_of_sales,
					dense_rank() over(partition by ci.city_name order by count(sl.sale_id) desc) as ranks
				from  sales as sl
					join customers as cu
						on sl.customer_id = cu.customer_id
					join city as ci
						on cu.city_id = ci.city_id
					join products as pr
						on pr.product_id = sl.product_id
				group by pr.product_name,ci.city_name
		)
	Select *
		from t1
		where t1.ranks <= 3;
```

7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products?
```sql
select 
		ci.city_name as city,
		count(distinct sl.customer_id) as counts
	from city as ci
		join customers as cu
			on ci.city_id = cu.city_id
		join sales as sl
			on cu.customer_id = sl.customer_id
		join products as pr
			on sl.product_id = pr.product_id
	where pr.product_id<=14
	group by ci.city_name;

	
select 
		ci.city_name as city,
		count(distinct sl.customer_id) as counts
	from city as ci
		join customers as cu
			on ci.city_id = cu.city_id
		join sales as sl
			on cu.customer_id = sl.customer_id
		join products as pr
			on sl.product_id = pr.product_id
	where sl.product_id in (1,2,3,4,5,6,7,8,9,10,11,12,13,14)
	group by ci.city_name
	order by counts desc;
```

8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer
```sql
with t1 as
		(
			select 
					ci.city_name as city,
					count(distinct sl.customer_id) as customer_count,
					sum(sl.total)::int/count(distinct(sl.customer_id)::int) As Avg_sale
				from city as ci
					join customers as cu
						on ci.city_id=cu.city_id
					join sales as sl
						on sl.customer_id = cu.customer_id
				group by ci.city_name
				order by City_name, Avg_sale
		),
	t2 as
		(
			select
					city_id,
					city_name,
					estimated_rent
				from city
		)
	select	
				t2.city_name,
				t2.estimated_rent,
				t1.customer_count,
				t1.avg_sale,
				Round((t2.estimated_rent/t1.customer_count)::numeric,2) as avg_rent_per_cx
			from t1
				join t2
					on t1.city=t2.city_name
			order by 4 desc;```sql

```

9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).
```sql
with monthly_sales as 
		(
			Select 
					ci.city_name as city,
					extract(month from sale_date) as months,
					extract(year from sale_date) as years,
					sum(sl.total) as total_sale
				from sales as sl
					join customers as cu
						on sl.customer_id = cu.customer_id
					join city as ci
						on cu.city_id = ci.city_id
						group by ci.city_name, months,years
						order by ci.city_name,years,months
		),
	growth as
		(
			select 
					city,
					months,
					years,
					total_sale as cur_sale,
					lag(total_sale,1)over(partition by city order by years, months) as prev_sale
				from monthly_sales
		)
	select  
			city,
			months,
			years,
			cur_sale,
			prev_sale,
			cur_sale-prev_sale as growth,
			Round(	
					(cur_sale-prev_sale)::numeric/prev_sale::numeric*100,2) as growth_rate
		from growth
		where prev_sale is not null;
```

10. **Market Potential Analysis**  
    Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated  coffee consumer
```sql
with t1 as 
		(
			select 
					cu.city_id as city,
					sum(sl.total) as total_sale,
					count(distinct cu.customer_id) as total_customer,
					count(sl.product_id) as coffe_consumer
			from sales as sl
				join products as pr
					on sl.product_id = pr.product_id
				join customers as cu
					on sl.customer_id = cu.customer_id
			where sl.product_id<=14	
			group by city
		),
	t2 as
		(
			select *,
					population*.25 as consumers
				from city
		)
	select 
			t2.city_name,
			t1.total_sale,
			sum(t2.estimated_rent) as total_rent,
			Round(t2.estimated_rent::numeric/total_customer::numeric,2) as avg_rent,
			t1.total_customer,
			t1.coffe_consumer,
			t2.consumers
		from t1 
			join t2
				on t1.city = t2.city_id
		group by t2.city_name,t1.total_sale,t1.total_customer,t1.coffe_consumer, t2.consumers,t2.estimated_rent
		order by t1.total_sale desc;
```
    

## Recommendations
After analyzing the data, the recommended top three cities for new store openings are:

**City 1: Pune**  
1. Average rent per customer is very low.  
2. Highest total revenue.  
3. Average sales per customer is also high.

**City 2: Delhi**  
1. Highest estimated coffee consumers at 7.7 million.  
2. Highest total number of customers, which is 68.  
3. Average rent per customer is 330 (still under 500).

**City 3: Jaipur**  
1. Highest number of customers, which is 69.  
2. Average rent per customer is very low at 156.  
3. Average sales per customer is better at 11.6k.

---

