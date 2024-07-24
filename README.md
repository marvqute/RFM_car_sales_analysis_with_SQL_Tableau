# RFM_car_sales_analysis_with_SQL_Tableau


---Inspecting the Data
select * from `sql-sandbox-422308.Sales.sales_data_sample`

--Checking for unique values
select distinct status from `sql-sandbox-422308.Sales.sales_data_sample`-
select distinct year_id from `sql-sandbox-422308.Sales.sales_data_sample`
select distinct PRODUCTLINE from `sql-sandbox-422308.Sales.sales_data_sample` 
select distinct COUNTRY from `sql-sandbox-422308.Sales.sales_data_sample`
select distinct DEALSIZE from `sql-sandbox-422308.Sales.sales_data_sample`
select distinct TERRITORY from `sql-sandbox-422308.Sales.sales_data_sample` 

select distinct MONTH_ID from `sql-sandbox-422308.Sales.sales_data_sample`
where year_id = 2003

---ANALYSIS
----Let's start by grouping sales by productline
select PRODUCTLINE, sum(sales) Revenue
from `sql-sandbox-422308.Sales.sales_data_sample`
group by PRODUCTLINE
order by 2 desc

-- revenue by year
select YEAR_ID, sum(sales) Revenue
from `sql-sandbox-422308.Sales.sales_data_sample`
group by YEAR_ID
order by 2 desc


--Revenue by deal size
select  DEALSIZE,  sum(sales) Revenue
from `sql-sandbox-422308.Sales.sales_data_sample`
group by  DEALSIZE
order by 2 desc


----What was the best month for sales in a specific year? How much was earned that month? 
select  MONTH_ID, sum(sales) Revenue, count(ORDERNUMBER) Frequency
from `sql-sandbox-422308.Sales.sales_data_sample`
where YEAR_ID = 2004 --change year to see the rest
group by  MONTH_ID
order by 2 desc


--November seems to be the month, what product do they sell in November, Classic I believe
select  MONTH_ID, PRODUCTLINE, sum(sales) Revenue, count(ORDERNUMBER)
from `sql-sandbox-422308.Sales.sales_data_sample`
where YEAR_ID = 2004 and MONTH_ID = 11 --change year to see the rest
group by  MONTH_ID, PRODUCTLINE
order by 3 desc


----Who is our best customer (this could be best answered with RFM)

DROP TABLE IF EXISTS `sql-sandbox-422308.Sales.#rfm`
CREATE TABLE `sql-sandbox-422308.Sales.Srfm` AS
WITH rfm AS 
(
  SELECT 
      CUSTOMERNAME, SUM(SALES) AS REVENUE, AVG(SALES) AS AVG_REVENUE,
      COUNT(ORDERNUMBER) AS COUNT, MAX(ORDERDATE) AS LAST_ORDER_DATE,
      (SELECT MAX(ORDERDATE) FROM `sql-sandbox-422308.Sales.sales_data_sample`) AS MAX_ORDER_DATE,
      DATE_DIFF ((SELECT MAX(ORDERDATE) FROM `sql-sandbox-422308.Sales.sales_data_sample`), MAX(ORDERDATE), DAY) AS RECENCY
  FROM `sql-sandbox-422308.Sales.sales_data_sample`
  GROUP BY CUSTOMERNAME
),
rfm_calc AS 
(
select r.*,
     NTILE(4) OVER (ORDER BY RECENCY DESC) AS rfm_recency,
     NTILE(4) OVER (ORDER BY COUNT) AS rfm_frequency,
     NTILE(4) OVER (ORDER BY REVENUE) AS rfm_monetary
from rfm AS r
) 
select c.*, rfm_recency+ rfm_frequency+ rfm_monetary as rfm_cell,
cast(rfm_recency as string) || cast(rfm_frequency as string) || cast(rfm_monetary  as string) AS rfm_cell_string
from rfm_calc AS c


select CUSTOMERNAME , rfm_recency, rfm_frequency, rfm_monetary, 
(case 
		when rfm_cell_string in ('111', '112' , '121', '122', '123', '132', '211', '212', '114', '141') then 'lost_customers'  --lost customers
		when rfm_cell_string in ('133', '134', '143', '244', '334', '343', '344', '144') then 'slipping away, cannot lose' -- (Big spenders who haven’t purchased lately) slipping away
		when rfm_cell_string in ('311', '411', '331') then 'new customers'
		when rfm_cell_string in ('222', '223', '233', '322') then 'potential customers'
		when rfm_cell_string in ('323', '333','321', '422', '332', '432') then 'active' --(Customers who buy often & recently, but at low price points)
		when rfm_cell_string in ('433', '434', '443', '444') then 'loyal'
	end) as rfm_segment
FROM `sql-sandbox-422308.Sales.Srfm`

--- what city has the highest number of sales in a specific country?
select CITY, sum(SALES) as REVENUE
from `sql-sandbox-422308.Sales.sales_data_sample`
where COUNTRY = 'USA' --change country to see the others
group by CITY
order by 2 desc

---what is the best product in the respective countries in terms of sales?
select COUNTRY, PRODUCTLINE
from (select COUNTRY, PRODUCTLINE, 
rank() over (partition by COUNTRY order by sum(SALES) desc)as rnk 
from `sql-sandbox-422308.Sales.sales_data_sample`
group by COUNTRY, PRODUCTLINE) as ranked
where rnk = 1         
