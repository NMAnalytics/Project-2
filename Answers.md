# Answers

The first thing I am going to do is change the data type of the sales and price_each column to a decimal instead of a float. So that there are not more than two decimals places. 

```sql
ALTER table sales_data_sample
alter COLUMN sales DECIMAL(19,2)

ALTER table sales_data_sample
alter COLUMN priceeach DECIMAL(19,2)
```
---
Now, I want to find out what is the highest selling product.  

```sql
SELECT PRODUCTLINE,sum(sales) as total_sales
FROM DBO.sales_data_sample
group by PRODUCTLINE
order by total_sales DESC 
```
|PRODUCTLINE|total_sales|
|---|---|
|Classic Cars|3919615.66|
|Vintage Cars|1903150.84|
|Motorcycles|1166388.34|
|Trucks and Buses|1127789.84|
|Planes|975003.57|
|Ships|714437.13|
|Trains|226243.47|

The classic cars are the highest selling and the trains are the lowest selling. 

---

Let's find the highest selling year next.   
```sql
SELECT year_ID, sum(sales) as Revenue, count (ORDERNUMBER) as Frequency 
from DBO.sales_data_sample
group by year_id
order by revenue DESC

```
|year_ID|Revenue|Frequency|
|---|---|---|
|2004|4724162.60|1345|
|2003|3516979.54|1000|
|2005|1791486.71|478|

2004 is the highest selling year. Looking at the data, the year 2005 is so much lower. Let's find out why.

---
I'm going to run this query to find all the months for 2005. 

```sql
SELECT month_ID, sum(sales) as revenue, count (ORDERNUMBER) as Frequency 
from DBO.sales_data_sample
where year_id = '2005'
group by MONTH_ID
order by revenue DESC
```
|month_ID|revenue|Frequency|
|---|---|---|
|5|457861.06|120|
|3|374262.76|106|
|2|358186.18|97|
|1|339543.42|99|
|4|261633.29|56|

The year 2005 only goes up to May and the years 2003 and 2004 go from January to December 

---
Next, I want to find what is the highest selling month in the year 2004.

```sql
SELECT month_ID, sum(sales) as revenue, count (ORDERNUMBER) as Frequency 
from DBO.sales_data_sample
where year_id = '2004'
group by MONTH_ID
order by revenue DESC
```
|month_ID|revenue|Frequency|
|---|---|---|
|11|1089048.01|301|
|10|552924.25|159|
|8|461501.27|133|
|12|372802.66|110|
|7|327144.09|91|
|9|320750.91|95|
|1|316577.42|91|
|2|311419.53|86|
|6|286674.22|85|
|5|273438.39|74|
|4|206148.12|64|
|3|205733.73|56|

The highest selling month is November almost double of October.   

---
Next, we will find the deal size 
```sql
SELECT DEALSIZE, sum(sales) as revenue 
from sales_data_sample
group by DEALSIZE
```

|DEALSIZE|revenue|
|---|---|
|Medium|6087432.24|
|Small|2643077.35|
|Large|1302119.26|

The most common deal size is medium. 

---

Now, I am going to do an RFM analysis. An RFM is a technique that is used to rank customers based on the recency, frequency, and monetary of their transitions to find out who is the best customer and who are your worst customers. Recency refers to the time since the customer's last purchase. Frequency refers to the number of purchases made by the customer. Monetary value refers to the amount of money spent by the customer.



First I am going to create a query to find the monetary, frequency, and most recent transaction. Using sum, avg, and count. Then use max to find the most recent date ordered and the last date on the data sheet. Then use date diff to find the recency in days. Then I use the Ntile function to divide the data, put them into buckets and rank them from 1 to 4.


Then use the cast function to change to data type to varchar. I can use this in the future if I want to make more specific categories to put each customer's data into. But for now,  I will create a temp table and categorize the sales data by high-value customers, medium value customers, and low-value customers based on the total ranking of monetary, frequency, and most recent transactions.



```sql
drop table if EXISTS #rfm
;with rfm as 
(
SELECT customername, 
        sum(sales) as total_monetary, 
        avg(sales) as avg_monetary ,
        count(ORDERNUMBER) as frequnecy,
        max(orderdate) as most_recent_order,
    (select Max(ORDERDATE) from dbo.sales_data_sample) as max_table_date,
    DATEDIFF(DD,max(orderdate),(select Max(ORDERDATE) from dbo.sales_data_sample)) as Recency
    from dbo.sales_data_sample
    group by CUSTOMERNAME
),
rfm_calc  as 
 (
    SELECT r.*,
        ntile(4)over(order by Recency DESC) as rfm_recency ,
        ntile(4)over(order by frequnecy) as rfm_frequnecy ,
        ntile(4)over(order by total_monetary) as rfm_monetary 
    from rfm as r
)
SELECT c.*, rfm_recency + rfm_frequnecy + rfm_monetary as rfm_cell,
CAST(rfm_recency as varchar) + CAST(rfm_frequnecy as varchar) +CAST(rfm_monetary as varchar) as rfm_string_cell
into #rfm 
from rfm_calc as c
```


```sql
SELECT CUSTOMERNAME, rfm_recency , rfm_frequnecy, rfm_monetary, rfm_cell,
case 
    when rfm_cell >= 9 then 'high_value_customer'
    when rfm_cell >= 6 then 'medium_value_customer'
    when rfm_cell >= 3 then 'low_value_customer'
 end 
 from #rfm
 ```
|CUSTOMERNAME|rfm_recency|rfm_frequnecy|rfm_monetary|rfm_cell|(No column name)|
|---|---|---|---|---|---|
|Boards &amp; Toys Co.|3|1|1|5|low_value_customer|
|Atelier graphique|2|1|1|4|low_value_customer|
|Auto-Moto Classics Inc.|3|1|1|5|low_value_customer|
|Microscale Inc.|2|1|1|4|low_value_customer|
|Royale Belge|3|1|1|5|low_value_customer|
|Bavarian Collectables Imports, Co.|1|1|1|3|low_value_customer|
|Double Decker Gift Stores, Ltd|1|1|1|3|low_value_customer|
|Cambridge Collectables Co.|1|1|1|3|low_value_customer|
|West Coast Collectables Co.|1|1|1|3|low_value_customer|
|Men &#39;R&#39; US Retailers, Ltd.|1|1|1|3|low_value_customer|
|CAF Imports|1|1|1|3|low_value_customer|
|Signal Collectibles Ltd.|1|1|1|3|low_value_customer|
|Mini Auto Werke|3|1|1|5|low_value_customer|
|Iberia Gift Imports, Corp.|1|1|1|3|low_value_customer|
|Online Mini Collectables|1|1|1|3|low_value_customer|
|Gift Ideas Corp.|3|1|1|5|low_value_customer|
|Clover Collections, Co.|1|1|1|3|low_value_customer|
|Australian Gift Network, Co|3|1|1|5|low_value_customer|
|Australian Collectables, Ltd|4|2|1|7|medium_value_customer|
|Auto Assoc. &amp; Cie.|1|1|1|3|low_value_customer|
|Classic Gift Ideas, Inc|2|2|1|5|low_value_customer|
|Osaka Souveniers Co.|1|2|1|4|low_value_customer|
|Daedalus Designs Imports|1|2|1|4|low_value_customer|
|Alpha Cognac|4|2|2|8|medium_value_customer|
|Diecast Collectables|1|1|2|4|low_value_customer|
|Quebec Home Shopping Network|4|2|2|8|medium_value_customer|
|Mini Wheels Co.|2|2|2|6|medium_value_customer|
|Royal Canadian Collectables, Ltd.|1|3|2|6|medium_value_customer|
|Marseille Mini Autos|3|2|2|7|medium_value_customer|
|Petit Auto|4|2|2|8|medium_value_customer|
|Canadian Gift Exchange Network|2|2|2|6|medium_value_customer|
|Volvo Model Replicas, Co|2|1|2|5|low_value_customer|
|Classic Legends Inc.|2|2|2|6|medium_value_customer|
|giftsbymail.co.uk|2|3|2|7|medium_value_customer|
|Enaco Distributors|2|2|2|6|medium_value_customer|
|Lyon Souveniers|4|2|2|8|medium_value_customer|
|Norway Gifts By Mail, Co.|1|2|2|5|low_value_customer|
|Super Scale Inc.|1|1|2|4|low_value_customer|
|Mini Caravy|4|1|2|7|medium_value_customer|
|Collectables For Less Inc.|3|2|2|7|medium_value_customer|
|Signal Gift Stores|3|3|2|8|medium_value_customer|
|Gifts4AllAges.com|4|2|2|8|medium_value_customer|
|Tekni Collectables Inc.|4|2|2|8|medium_value_customer|
|Motor Mint Distributors Inc.|2|2|2|6|medium_value_customer|
|Blauer See Auto, Co.|2|2|2|6|medium_value_customer|
|Mini Classics|2|3|2|7|medium_value_customer|
|Collectable Mini Designs Co.|1|2|3|6|medium_value_customer|
|Vitachrome Inc.|2|2|3|7|medium_value_customer|
|Stylish Desk Decors, Co.|3|3|3|9|high_value_customer|
|Auto Canal Petit|4|3|3|10|high_value_customer|
|Cruz &amp; Sons Co.|2|3|3|8|medium_value_customer|
|Amica Models &amp; Co.|1|3|3|7|medium_value_customer|
|La Corne D&#39;abondance, Co.|2|2|3|7|medium_value_customer|
|FunGiftIdeas.com|3|3|3|9|high_value_customer|
|Toms Spezialitten, Ltd|2|3|3|8|medium_value_customer|
|Heintze Collectables|2|3|3|8|medium_value_customer|
|Gift Depot Inc.|4|2|3|9|high_value_customer|
|Marta&#39;s Replicas Co.|1|3|3|7|medium_value_customer|
|Oulu Toy Supplies, Inc.|3|3|3|9|high_value_customer|
|Toys4GrownUps.com|3|3|3|9|high_value_customer|
|Mini Creations Ltd.|3|4|3|10|high_value_customer|
|Toys of Finland, Co.|3|3|3|9|high_value_customer|
|Herkku Gifts|1|3|3|7|medium_value_customer|
|Suominen Souveniers|3|3|3|9|high_value_customer|
|Handji Gifts&amp; Co|4|4|3|11|high_value_customer|
|Baane Mini Imports|2|3|3|8|medium_value_customer|
|Vida Sport, Ltd|1|3|3|7|medium_value_customer|
|UK Collectables, Ltd.|4|3|3|10|high_value_customer|
|Tokyo Collectables, Ltd|4|3|3|10|high_value_customer|
|Corrida Auto Replicas, Ltd|2|3|4|9|high_value_customer|
|Technics Stores Inc.|3|4|4|11|high_value_customer|
|Diecast Classics Inc.|4|3|4|11|high_value_customer|
|Online Diecast Creations Co.|2|4|4|10|high_value_customer|
|Scandinavian Gift Ideas|3|4|4|11|high_value_customer|
|Reims Collectables|4|4|4|12|high_value_customer|
|Rovelli Gifts|2|4|4|10|high_value_customer|
|L&#39;ordine Souveniers|4|4|4|12|high_value_customer|
|Saveley &amp; Henriot, Co.|1|4|4|9|high_value_customer|
|Danish Wholesale Imports|4|4|4|12|high_value_customer|
|Salzburg Collectables|4|4|4|12|high_value_customer|
|Corporate Gift Ideas Co.|3|4|4|11|high_value_customer|
|Souveniers And Things Co.|4|4|4|12|high_value_customer|
|Anna&#39;s Decorations, Ltd|3|4|4|11|high_value_customer|
|AV Stores, Co.|2|4|4|10|high_value_customer|
|The Sharp Gifts Warehouse|4|4|4|12|high_value_customer|
|Land of Toys Inc.|2|4|4|10|high_value_customer|
|Dragon Souveniers, Ltd.|3|4|4|11|high_value_customer|
|La Rochelle Gifts|4|4|4|12|high_value_customer|
|Muscle Machine Inc|3|4|4|11|high_value_customer|
|Australian Collectors, Co.|3|4|4|11|high_value_customer|
|Mini Gifts Distributors Ltd.|4|4|4|12|high_value_customer|
|Euro Shopping Channel|4|4|4|12|high_value_customer|


Now I know who my best customers are and which customers to market towards to increase sales. Companies like Euro Shopping Channel are high-value customers because they buy a lot. When they buy products they spend a lot and they have bought recently. Also, I  can target companies that have spent a lot of money but have not bought products recently to help increase sales. Another way is to target companies that have bought a lot but don't spend a lot of money.   

---

The last analysis I will do is to find out what products are sold together the most. First I will group the order numbers and count all of the order numbers that have shipped. Next by using -where we can search for the number of products in each order for any order number. Next put the data into an XML to have them all in one column. After that, use the -stuff function to get rid of the commas and turn it into a string. 


Now we join both tables together and order by product code. I can now use that data to look for products that were bought together. 







```sql
select distinct top 20 ordernumber ,stuff(
(SELECT ',' + productcode 
from sales_data_sample as og  
where ordernumber in 
(
    SELECT ordernumber 
    from   
        (
            SELECT ordernumber, count(*) as number_of_product_in_each_order
            from sales_data_sample 
            where status = 'Shipped'
            group by ORDERNUMBER
        )
        m 
    where number_of_product_in_each_order = '6'
)
and og.ordernumber  = sd.ordernumber
for xml path (''))
,1, 1, '') as product_code
from sales_data_sample as sd
order by 2 DESC
```
  |ordernumber|product_code|
|---|---|
|10156|S50_1341,S700_1691|
|10385|S24_3816,S700_1138|
|10154|S24_3151,S700_2610|
|10146|S18_3782,S18_4721|
|10290|S18_3320,S24_4258|
|10323|S18_3320,S18_4600|
|10265|S18_3278,S18_3482|
|10130|S18_3029,S18_3856|
|10269|S18_2957,S24_4258|
|10255|S18_2795,S24_2022|
|10243|S18_2325,S24_1937|
|10409|S18_2325,S24_1937|
|10303|S18_2248,S24_3969|
|10125|S18_1342,S18_2795|
|10102|S18_1342,S18_1367|
|10256|S18_1342,S18_1367|
|10231|S12_1108,S12_3891|
|10298|S10_2016,S18_2625|
|10112|S10_1949,S18_2949|
|10100|NULL|



I can see that order numbers 10102 and 10256 have the same products that were bought together, so using this query is a very helpful way to find other orders that have similar products that are bought together. 
