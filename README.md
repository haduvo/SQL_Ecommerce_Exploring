# SQL_Ecommerce_Exploring
## Exploring E_Commerce by Bigquery base on Google Analytics dataset
The dataset is stored in a public Google BigQuery dataset. Data contains information about user sessions on a website collected from Google Analytics in 2017.
Author uses Google BigQuery tool to write SQL queries and have an outlook about business situation.
## 1. Calculate total visit, pageview, transaction for Jan, Feb and March 2017:
```
with getdata as (SELECT date
      , totals.visits
      , totals.pageviews
      , totals.transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
where _table_suffix between '0101' and '0331')
select left(date,6) as month
      , sum(visits) as visits
      , sum(pageviews) as pagevies
      , sum(transactions) as transactions
from getdata
group by month
order by month;
```
## 2. Bounce rate per traffic source in July 2017:
```
with traffic_source as (SELECT 
       trafficSource.source
      , totals.bounces
      , totals.visits
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
where _table_suffix between '01' and '31')
select source
      , sum(visits) as total_visits
      , sum(bounces) as total_no_of_bounces
      , (sum(bounces))*100/ (sum(visits))
      from traffic_source
group by source
order by total_visits DESC;
```
## 3. Revenue by traffic source by week, by month in June 2017:
```
with week as (SELECT format_date ('%Y%W', parse_date('%Y%m%d',date) ) as time
      , trafficSource.source
      , productRevenue as revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product),
mth as (SELECT format_date ('%Y%m', parse_date('%Y%m%d',date) ) as time
      , trafficSource.source
      , productRevenue as revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product)
select 'Week' as time_type
      , week.time
      , source
      ,sum(revenue) as revenue
from week
group by week.time, source
union all
select 'Month' as time_type
      , mth.time
      , source
      ,sum(revenue) as revenue
from mth
group by mth.time, source
order by revenue DESC;
```
## 4. Average number of pageviews by purchase type in June, July 2017:
```
with pur as (SELECT format_date('%Y%m',parse_date('%Y%m%d',date) ) as Month
      , count(distinct (fullVisitorId)) as purchase
      , sum (totals.pageviews) as pageviews
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits,
      UNNEST (hits.product) product
      where _table_suffix between '0601' and '0731' and productRevenue is not null and totals.transactions >= 1
group by month)
, nonpur as (SELECT format_date('%Y%m',parse_date('%Y%m%d',date) ) as Month
      , count(distinct (fullVisitorId)) as nonpurchase
      , sum (totals.pageviews) as pageviews
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits,
      UNNEST (hits.product) product
      where _table_suffix between '0601' and '0731' and productRevenue is null and totals.transactions is null
group by month)
select pur.Month
      , pur.pageviews/ pur.purchase as avg_pageviews_purchase
      , nonpur.pageviews/ nonpur.nonpurchase as avg_pageviews_non_purchase
from pur
left join nonpur using (month)
order by pur.Month;
```
## 5. Average number of transactions per user that made a purchase in July 2017:
```
with num_trans as (SELECT format_date('%Y%m',parse_date('%Y%m%d',date) ) as Month
      , sum(totals.transactions) as totals_transactions
      , count (distinct(fullVisitorId)) as transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
      UNNEST (hits.product) product
      where productRevenue is not null and totals.transactions >= 1
group by month)
select  Month
      , (totals_transactions/transactions_per_user) as Avg_total_transactions_per_user
      FROM num_trans
group by month, totals_transactions, transactions_per_user;
```
## 6. Avarage amount of money spent per session. Only include purchaser data in July 2017:
```
with money as (SELECT format_date('%Y%m',parse_date('%Y%m%d',date) ) as Month
      , count(productRevenue) as total_visit
      , sum(productRevenue) as total_revenue
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
      UNNEST (hits.product) product
      where productRevenue is not null and totals.transactions >= 1
group by month)
select month 
      ,total_revenue/total_visit as avg_revenue_by_user_per_visit
from money;
```
## 7. Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017:
```
with rd1 as (SELECT  
       distinct fullVisitorId as id
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
      UNNEST (hits.product) product
      where  v2ProductName like 'YouTube Men_s Vintage Henley' and productRevenue is not null) 
, rd2 as (select productQuantity
      ,v2ProductName
      , fullVisitorId
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
      UNNEST (hits.product) product
      where v2ProductName not like 'YouTube Men_s Vintage Henley' and productRevenue is not null)
select rd2.v2ProductName as other_purchased_products
      , sum (productQuantity) as quantity
from rd1
left join rd2 on rd1.id = rd2.fullVisitorId
where rd2.fullVisitorId = rd1.id 
group by other_purchased_products
order by quantity DESC;
```
## 8. Calculate cohort map from product view to addocart to purchase in Jan, Feb and March 2017:
```
with num_view as (SELECT format_date ('%Y%m', parse_date('%Y%m%d',date) ) as Month
      , eCommerceAction.action_type
      , count (case when eCommerceAction.action_type = '2' then eCommerceAction.action_type else null end) as num_product_view
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits
where _table_suffix between '0101' and '0331' and eCommerceAction.action_type = '2'
group by month, eCommerceAction.action_type
order by month)
, num_add as (SELECT format_date ('%Y%m', parse_date('%Y%m%d',date) ) as Month
      , eCommerceAction.action_type
      , count (case when eCommerceAction.action_type = '3' then eCommerceAction.action_type else null end) as num_addtocart
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits
where _table_suffix between '0101' and '0331' and eCommerceAction.action_type = '3'
group by month, eCommerceAction.action_type
order by month)
, num_pur as (SELECT format_date ('%Y%m', parse_date('%Y%m%d',date) ) as Month
      , eCommerceAction.action_type
      , count (case when eCommerceAction.action_type = '6' then eCommerceAction.action_type else null end) as num_purchase
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
where _table_suffix between '0101' and '0331' and eCommerceAction.action_type = '6' and productRevenue is not null
group by month, eCommerceAction.action_type
order by month)
select num_view.month
      , num_view.num_product_view
      , num_add.num_addtocart
      , num_pur.num_purchase
      , (num_add.num_addtocart/num_view.num_product_view) as add_to_cart_rate
      , (num_pur.num_purchase/num_view.num_product_view) as purchase_rate
from num_view
full join num_add using (month)
full join num_pur using (month)
group by month, num_product_view, num_addtocart, num_purchase
order by month;
```

