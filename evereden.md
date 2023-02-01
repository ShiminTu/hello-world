1. 将上述两张excel sheets 导入数据库, 注意数据格式  eg：支付时间 应为 datetime  

我使用的是bigquery。

2. 筛选订单状态为“已签收”的订单

```
SELECT  *
FROM `summer-reef-327404.223344.ee_order`
WHERE status = '已签收'
```

3. 价位段分层：基于每位顾客在这段时间的总订单金额  

-  3.1 请计算每个月各价位段(￥0-100, ￥100-200, ￥200-400, >￥400）中顾客人数及占比  
```
with agg_t as (
  select FORMAT_DATE('%Y-%m', order_time) as month, phone_num, sum(total_amt) as monthly_amt
  from `summer-reef-327404.223344.ee_order`
  group by FORMAT_DATE('%Y-%m', order_time), phone_num
)

select t1.month, t1.level, t1.cust_cnt, concat(round(t1.cust_cnt/t2.all_cust*100,2),'%') as ppt
from (
  select month,level,count(phone_num) as cust_cnt
  from (
    select month,phone_num,  CASE 
      WHEN monthly_amt <= 100 THEN '¥0-100'
      WHEN monthly_amt > 100 and monthly_amt <= 200 THEN '¥100-200'
      WHEN monthly_amt > 200 and monthly_amt <= 400 THEN '¥200-400'
      ELSE '>¥400'
      END
      AS level
    from agg_t
  ) tt
group by month,level
)t1
left join (
  select month, count(phone_num) as all_cust
  from agg_t
  group by month
)t2
on t1.month = t2.month
order by month,level
```
- 3.2 请计算各类目下各价位段中顾客人数及占比

```
with newt as (
  SELECT t1.phone_num,t2.item_cate, sum(total_amt) as all_amt
  FROM `summer-reef-327404.223344.ee_order` as t1
  inner join `summer-reef-327404.223344.ee_cat` as t2 #order表有行item_name为null
  on t1.item_name = t2.item_name
  group by t1.phone_num,t2.item_cate
)

select t1.item_cate, t1.level, t1.cust_cnt, concat(round(t1.cust_cnt/t2.all_cust*100,2),'%') as pct
from (
  select item_cate,level,count(phone_num) as cust_cnt
  from (
    select item_cate,phone_num,  CASE 
      WHEN all_amt <= 100 THEN '¥0-100'
      WHEN all_amt > 100 and all_amt <= 200 THEN '¥100-200'
      WHEN all_amt > 200 and all_amt <= 400 THEN '¥200-400'
      ELSE '>¥400'
      END
      AS level
    from newt
  ) tt
group by item_cate,level
)t1
left join (
  select item_cate, count(phone_num) as all_cust
  from newt
  group by item_cate
)t2
on t1.item_cate = t2.item_cate
order by item_cate,level
```

4. 如果定义在本段时间内只有一次支付记录的顾客 为 新客，有 >=2 次支付记录的顾客 为 老客,

- 4.1 请展示出每位老客 上一次订单 的支付时间

```
SELECT phone_num, max(order_time)
FROM `summer-reef-327404.223344.ee_order` 
where phone_num in (
  select phone_num
  from `summer-reef-327404.223344.ee_order` 
  group by phone_num, order_time
  having count(*) >=2
)
group by phone_num
```


- 4.2 请计算每个月的老客占比(本月老客/本月总顾客数)  

**数据时间跨度为2019-01至2019-3月，根据定义，在2019-01有过两次购买记录的顾客在2019-01也被定义为老客。**

```
with newt as (
  select phone_num, format_date('%Y-%m-%d', order_time) as order_date
  from `summer-reef-327404.223344.ee_order`
)

select month, concat(round(sum(if_old)/count(if_old)*100,2),'%') as pct
from (
select *, row_number() over (partition by phone_num,month order by if_old desc) as ranking
from (
SELECT phone_num, order_date, left(order_date,7) as month,
  (case when exists (select * from newt where a.phone_num = phone_num and order_date < a.order_date) then 1 else 0 end) as if_old
FROM newt as a
) t1 ) t2
where ranking = 1
group by month
order by month

```

