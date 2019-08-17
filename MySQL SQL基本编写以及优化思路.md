[TOC]



# MySQL SQL基本编写以及优化思路

## 主要内容

GROUP BY ，DISTINCT

COUNT，MIN，MAX，SUM，AVG

ORDER BY

HAVING

SUBQUERY种类

行转列，列转行

### GROUP BY

作用：

```powershell
1 去掉重复值
2 对group by的关键字，进行排序（8.0开始不排序），根据hex()转换的ASC码排序！NULL排在前列！
注意：在5.7版本中可以在group by后面添加 order by  null去掉排序
```

实验：

```sql
mysql> flush status;
Query OK, 0 rows affected (0.00 sec)

mysql> show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 0     |
+-------------------------+-------+
3 rows in set (0.00 sec)

mysql> select dept_no from t_group group by dept_no;
+---------+
| dept_no |
+---------+
| d002    |
| d004    |
| d005    |
| d006    |
| d007    |
| d008    |
+---------+
6 rows in set (0.00 sec)

mysql> show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 0     |
| Created_tmp_tables      | 1     |
+-------------------------+-------+
3 rows in set (0.00 sec)

mysql> show status like '%sort%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Sort_merge_passes | 0     |
| Sort_range        | 0     |
| Sort_rows         | 6     |
| Sort_scan         | 1     |
+-------------------+-------+
4 rows in set (0.00 sec)

mysql> flush status;
Query OK, 0 rows affected (0.00 sec)
--可以看到group by进行了排序

--添加order by null去掉排序：
mysql> select dept_no from t_group group by dept_no order by null;
+---------+
| dept_no |
+---------+
| d006    |
| d005    |
| d002    |
| d008    |
| d007    |
| d004    |
+---------+
6 rows in set (0.00 sec)

mysql> show status like '%sort%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Sort_merge_passes | 0     |
| Sort_range        | 0     |
| Sort_rows         | 0     |
| Sort_scan         | 0     |
+-------------------+-------+
4 rows in set (0.00 sec)


***************************************************************************************************

--如何使用order by 排序将NULL放在最后：
mysql> select a from ( select 900 a union all select 1 a union all select 20 a union all select null a union all select 8 a union all select 7 a) c  order by ifnull(a,9999);
+------+
| a    |
+------+
|    1 |
|    7 |
|    8 |
|   20 |
|  900 |
| NULL |
+------+

--注意：数字隐式转成字符，排序有误！
mysql> select a from ( select 900 a union all select 1 a union all select 20 a union all select null a union all select 8 a union all select 7 a) c  order by ifnull(a,'9999');
+------+
| a    |
+------+
|    1 |
|   20 |
|    7 |
|    8 |
|  900 |
| NULL |
+------+
6 rows in set (0.00 sec)

***************************************************************************************************
group by 使用索引，减少using tempfile 、Using temporary的两种方式
1.LOOSE INDEX SCAN：
  
索引:(dept_no,to_date)
mysql> desc select dept_no,to_date from t_group5 group by dept_no,to_date;
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | t_group5 | NULL       | range | ix_empno_to_date | ix_empno_to_date | 15      | NULL |    7 |   100.00 | Using index for group-by |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+

--distinct 联合索引前导列
mysql> desc select distinct dept_no,to_date from t_group5 ;
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | t_group5 | NULL       | range | ix_empno_to_date | ix_empno_to_date | 15      | NULL |    7 |   100.00 | Using index for group-by |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)

--没有where条件，group by索引前导列，其他列聚合函数
mysql> desc select dept_no,max(to_date) from t_group5 group by dept_no;
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | t_group5 | NULL       | range | ix_empno_to_date | ix_empno_to_date | 12      | NULL |    6 |   100.00 | Using index for group-by |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
--有where条件，含有索引前导列的范围条件，group by子句按照索引列顺序
mysql> desc select dept_no from t_group5 where dept_no like 'd%' group by dept_no;
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+---------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                                 |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t_group5 | NULL       | range | ix_empno_to_date | ix_empno_to_date | 12      | NULL |    3 |   100.00 | Using where; Using index for group-by |
+----+-------------+----------+------------+-------+------------------+------------------+---------+------+------+----------+---------------------------------------+


2.Tight Index Scan /range scan /index scan
  
--创建索引的时候，必须按照where条件后含有等号的列 加上 group by 后面的列的顺序来创建索引！
mysql> desc select dept_no,to_date,emp_no from t_group5 where to_date='9999-01-01' group by dept_no,emp_no;
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+----------------------------------------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra                                        |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+----------------------------------------------+
|  1 | SIMPLE      | t_group5 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 2448384 |    10.00 | Using where; Using temporary; Using filesort |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+----------------------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> create index idx_t2 on t_group5(to_date,dept_no,emp_no);
Query OK, 0 rows affected (19.07 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc select dept_no,to_date,emp_no from t_group5 where to_date='9999-01-01' group by dept_no,emp_no;
+----+-------------+----------+------------+------+---------------+--------+---------+-------+---------+----------+--------------------------+
| id | select_type | table    | partitions | type | possible_keys | key    | key_len | ref   | rows    | filtered | Extra                    |
+----+-------------+----------+------------+------+---------------+--------+---------+-------+---------+----------+--------------------------+
|  1 | SIMPLE      | t_group5 | NULL       | ref  | idx_t2        | idx_t2 | 3       | const | 1224192 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+------+---------------+--------+---------+-------+---------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```



#### 不完全group by 随着执行计划不同导致结果不同

```sql
mysql> select t.* ,count(1) from t_order t group by t.dept_no;
ERROR 1055 (42000): Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'employees.t.emp_no' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

mysql5.7默认不允许不完全group by
```



#### GROUP BY + 聚合函数+ distinct

```sql
mysql> select dept_no,count(a.to_date)
    -> from
    -> (select dept_no ,to_date from t_order group by dept_no,to_date) a 
    -> group by dept_no;
+---------+------------------+
| dept_no | count(a.to_date) |
+---------+------------------+
| d002    |                1 |
| d004    |                1 |
| d005    |                2 |
| d006    |                1 |
| d007    |                1 |
| d008    |                2 |
+---------+------------------+
6 rows in set (0.00 sec)
--分析：子查询按照dept_no 和to_date分组去重；外层查询再依据dept_no分组去重

mysql> select dept_no,count(distinct(to_date)) from t_order group by dept_no;
+---------+--------------------------+
| dept_no | count(distinct(to_date)) |
+---------+--------------------------+
| d002    |                        1 |
| d004    |                        1 |
| d005    |                        2 |
| d006    |                        1 |
| d007    |                        1 |
| d008    |                        2 |
+---------+--------------------------+
6 rows in set (0.00 sec)

--外层直接对to_date进行distinct去重，接着使用dept_no进行分组去重
```



#### Group BY相关性能问题案例分析

先聚合后join来减少join次数达到优化目的！

案例：

```mysql
eg1:
优化前：
select l.liquidactor_id as liquidactorid,
       l.liquidactor_name as liquidactorName,
	   s.store_id as storeid,
	   s.merchant_name as  storeName,
	   sum(o.real_money) as totalrealmoney,
	   sum(o.net_money) as totalnetmoney,
	   sum(o.bank_fee) as totalbankfee,
	   sum(o.real_money-o.pay_platform_fee) as withdrawmoney,
	   o.pay_day as createday,
	   (unix_timestamp(now())*1000),
	   (unix_timestamp(now())*1000)
from lp_finance o
left join lp_liquidactor l on o.liquidactor_id=l.liquidactor_id
left join lp_liquidactor_store s on o.store_id=s.store_id
where ((o.pay_day='20171227') and (o.pay_status=1))
group by s.store_id;

优化后：
select l.liquidactor_id as liquidactorid,
       l.liquidactor_name as liquidactorName,
	   s.store_id as storeid,
	   s.merchant_name as  storeName,
	   sum(o.real_money) as totalrealmoney,
	   sum(o.net_money) as totalnetmoney,
	   sum(o.bank_fee) as totalbankfee,
	   sum(o.real_money-o.pay_platform_fee) as withdrawmoney,
	   o.pay_day as createday,
	   (unix_timestamp(now())*1000),
	   (unix_timestamp(now())*1000) from
(select o.store_id,
	   o.liquidactor_id,
	   sum(o.real_money) as totalrealmoney,
	   sum(o.net_money) as totalnetmoney,
	   sum(o.bank_fee) as totalbankfee,
	   sum(o.real_money-o.pay_platform_fee) as withdrawmoney,
	   o.pay_day as createday,
	   (unix_timestamp(now())*1000),
	   (unix_timestamp(now())*1000)
from lp_finance o force index(IDX_lQID_SID)
where ((o.pay_day='20171227') and (o.pay_status=1))
group by o.store_id,o.liquidactor_id   ----将group by提前减少join的次数
) o
left join lp_liquidactor l on o.liquidactor_id=l.liquidactor_id
left join lp_liquidactor_store s on o.store_id=s.store_id
where ((o.pay_day='20171227') and (o.pay_status=1))
group by o.store_id;



eg2:
select eina.infnr as infnr,
       eina.lifnr,
	   eine.werks,
	   eina.matnr,
	   t.name as lifnrname
from 
       erp_eina eina
left join erp_eine eine on eine.infnr=eina.infnr
left join rep_lftl t on eina.lifnr=t.lifnr
inner join scm.demand_pool d on d.supplier_no=eina.lifnr and d.goods_no=eina.matnr
where t.loevn='0'
and eine.loekz='0'
and eine.werks in ('asc','B88','301','ppp')
group by 
eina.lifnr,eine.werks,eina.matnr;


经过查询表的rows大小，d表rows最大，把d表摘除，放在group by过滤后，join，用来减少join的次数
select distinct(eina.infnr) as infnr,
       eina.lifnr,
	   eina.werks,
	   eina.matnr,
	   eina.name as lifnrname
from
(select eina.infnr,
       eina.lifnr,
	   eine.werks,
	   eina.matnr,
	   t.name 
from 
erp_eina eina
left join erp_eine eine on eine.infnr=eina.infnr
left join rep_lftl t on eina.lifnr=t.lifnr
where t.loevn='0'
and eine.loekz='0'
and eine.werks in ('asc','B88','301','ppp')
group by 
eina.lifnr,eine.werks,eina.matnr) eina
inner join scm.demand_pool d on d.supplier_no=eina.lifnr and d.goods_no=eina.matnr;
在根据执行计划，添加d表过滤条件的索引：
create index ix_d on demand_pool(goods_no,supplier_no)
```



### left join 

不满足多对一，易导致结果出错

left join需要满足N：1原则，右边的表（即右边表的连接列不能有重复值）和左边的连接条件保证唯一性

```SQL
mysql> select * from departments;
+---------+--------------------+
| dept_no | dept_name          |
+---------+--------------------+
| d009    | Customer Service   |
| d005    | Development        |
| d002    | Finance            |
| d003    | Human Resources    |
| d001    | Marketing          |
| d004    | Production         |
| d006    | Quality Management |
| d008    | Research           |
| d007    | Sales              |
+---------+--------------------+
9 rows in set (0.05 sec)

mysql> select * from t_order order by 2;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|   NULL | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
+--------+---------+------------+------------+
10 rows in set (0.06 sec)

mysql> select * from t_order t left join departments d on t.dept_no=d.dept_no order by 2;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|   NULL | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | d008    | Research           |
+--------+---------+------------+------------+---------+--------------------+
10 rows in set (0.00 sec)

右边表中有重复值，连接之后结果，产生多行重复值
mysql> select * from departments d left join t_order t on t.dept_no=d.dept_no order by 2;
+---------+--------------------+--------+---------+------------+------------+
| dept_no | dept_name          | emp_no | dept_no | from_date  | to_date    |
+---------+--------------------+--------+---------+------------+------------+
| d009    | Customer Service   |   NULL | NULL    | NULL       | NULL       |
| d005    | Development        |  40983 | d005    | 1986-12-01 | 9999-01-01 |
| d005    | Development        |  24007 | d005    | 1986-12-01 | 9999-01-01 |
| d005    | Development        |  50449 | d005    | 1986-12-01 | 9999-01-01 |
| d005    | Development        |  30970 | d005    | 1986-12-01 | 2017-03-29 |
| d002    | Finance            |  31112 | d002    | 1986-12-01 | 1993-12-10 |
| d003    | Human Resources    |   NULL | NULL    | NULL       | NULL       |
| d001    | Marketing          |   NULL | NULL    | NULL       | NULL       |
| d004    | Production         |  10004 | d004    | 1986-12-01 | 9999-01-01 |
| d006    | Quality Management |  22744 | d006    | 1986-12-01 | 9999-01-01 |
| d008    | Research           |  48317 | d008    | 1986-12-01 | 1989-01-11 |
| d008    | Research           |   NULL | d008    | 1986-12-01 | 1992-05-27 |
| d007    | Sales              |  49667 | d007    | 1986-12-01 | 9999-01-01 |
+---------+--------------------+--------+---------+------------+------------+
--而产生多行重复值之后，开发人员使用group by去重，使用的不完全group by
mysql> select * from departments d left join t_order t on t.dept_no=d.dept_no group by d.dept_no order by 2;
ERROR 1055 (42000): Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'employees.t.emp_no' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

解决方法：
保证右边表的连接条件的唯一性！
mysql> select * from departments d left join (select dept_no,min(emp_no) emp_no,min(from_date) date,min(to_date) to_date from t_order group by dept_no) t on t.dept_no=d.dept_no order by 1;
+---------+--------------------+---------+--------+------------+------------+
| dept_no | dept_name          | dept_no | emp_no | date       | to_date    |
+---------+--------------------+---------+--------+------------+------------+
| d001    | Marketing          | NULL    |   NULL | NULL       | NULL       |
| d002    | Finance            | d002    |  31112 | 1986-12-01 | 1993-12-10 |
| d003    | Human Resources    | NULL    |   NULL | NULL       | NULL       |
| d004    | Production         | d004    |  10004 | 1986-12-01 | 9999-01-01 |
| d005    | Development        | d005    |  24007 | 1986-12-01 | 2017-03-29 |
| d006    | Quality Management | d006    |  22744 | 1986-12-01 | 9999-01-01 |
| d007    | Sales              | d007    |  49667 | 1986-12-01 | 9999-01-01 |
| d008    | Research           | d008    |  48317 | 1986-12-01 | 1989-01-11 |
| d009    | Customer Service   | NULL    |   NULL | NULL       | NULL       |
+---------+--------------------+---------+--------+------------+------------+


```



### order by

基本用法：接列名，接数字表示第几列（最好少用，防止表结构变化）。

高级用法：使得特定的行排在特定的位置！

```SQL
mysql> select * from t_order order by case when dept_no='d008' then '1' else dept_no end asc,emp_no desc;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|   NULL | d008    | 1986-12-01 | 1992-05-27 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
注意：order by排序需要的资源要看select当中所有的列，不仅仅只看order by的列。所以选择不必要的列或者*会印象SQL性能！

```

#### order by 、group by的一些用法

```sql
--要求显示t_order表中dept_no当中最大的emp_no的所有信息：
mysql> select * from t_order;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|   NULL | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.05 sec)

mysql> set session sql_mode='';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select * from (select * from t_order order by dept_no asc,emp_no desc) t group by t.dept_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|   NULL | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
6 rows in set (0.00 sec)

mysql> desc select * from (select * from t_order order by dept_no asc,emp_no desc) t group by t.dept_no;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | t_order | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using temporary; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> show warnings\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t_order`.`emp_no` AS `emp_no`,`employees`.`t_order`.`dept_no` AS `dept_no`,`employees`.`t_order`.`from_date` AS `from_date`,`employees`.`t_order`.`to_date` AS `to_date` from `employees`.`t_order` group by `employees`.`t_order`.`dept_no`
1 row in set (0.00 sec)

ERROR: 
No query specified
5.7关闭sql_mode 之后，优化器打开了子查询导致结果未达到预期，添加limit，使得优化器不打开子查询
mysql> select * from (select * from t_order order by dept_no asc,emp_no desc limit 100) t group by t.dept_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
+--------+---------+------------+------------+
6 rows in set (0.00 sec)

8.0和Oracle当中可以使用row_number()函数
mysql> select s.* from (select t.*,row_number() over(partition by t.dept_no order by t.emp_no desc) rn from t_order t) s where s.rn=1;

利用字符串的比较，从左到右的比较方式，避免不完全group by：
mysql> select max(emp_no) emp_no,t.dept_no,
    -> substr(max(concat(lpad(emp_no,5,'0'),from_date)),6) from_date,
    -> substr(max(concat(lpad(emp_no,5,'0'),to_date)),6) to_date
    -> from t_order t
    -> group by t.dept_no  order by 2;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
+--------+---------+------------+------------+
6 rows in set (0.00 sec)

```



#### order by 内部算法

```SQL
single pass:
一次性把SQL中涉及到的字段全部读出来，然后依据排序字段排序，最后直接返回结果
优点：只需一次IO，降低IO开销
缺点：内存空间不够，在磁盘排序

two pass
先读取行指针和排序字段，进行排序，而后依据排序结果再去读取所需数据
优点：排序量小
缺点：读取时，会发生大量IO

mysql> show status like '%sort%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Sort_merge_passes | 0     |
| Sort_range        | 0     |
| Sort_rows         | 12    |
| Sort_scan         | 2     |
+-------------------+-------+
4 rows in set (0.00 sec)

排序总字段长度超过如下阈值：
mysql> show variables like '%max_length%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| max_length_for_sort_data | 1024  |
+--------------------------+-------+
1 row in set (0.01 sec)

select涉及到的字段中含有blob text的类型，会触发two pass 排序

注意：order by ... limit   order by 的列， 有重复值导致分页后数据有问题
解决方法： 1 在order by后面添加PK 或者能区分重复值的列排序
          2 出现的原因是order + limit没有排序完成!  
```



### update

mysql当中update where子句不能使用子查询，优化思路使用join来实现：

```sql
mysql当中update where子句不能使用子查询：
update state join (select max(token.openid) openid,token.phone ,v.vin from vei v,user_token token where token.id=v.user_id group by token.phone,v.vin) m
on state.phone=m.phone and state.vin=m.vin 
set state.wechat_user=m.openid;

错误的sql语句：
update tableA set name='myname' where id in (select aid from tableB where age=25);
正确的sql语句：
update tableA as a inner join (select aid from tableB where age=25) as b set name='myname'
where a.id=b.aid;
```



### count

基本用法： 统计数量 ,count(col)不计算NULL

```sql
mysql> select count(emp_no) from t_order;
+---------------+
| count(emp_no) |
+---------------+
|             9 |
+---------------+
1 row in set (0.06 sec)

mysql> select count(*) from t_order;
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.00 sec)
```



### MIN/MAX

求出最小值，最大值，不计NULL

字符类型正常比较 ，不计NULL

还可以验证该字段是否有指定值，若有返回1，没有返回0：

```SQL
--emp_no=1000不存在
mysql> select min(emp_no) from t_order where emp_no=1000;
+-------------+
| min(emp_no) |
+-------------+
|        NULL |
+-------------+
1 row in set (0.00 sec)

mysql> select min(emp_no) from t_order where emp_no=10004;
+-------------+
| min(emp_no) |
+-------------+
|       10004 |
+-------------+
1 row in set (0.00 sec)

--通过case when来判断字段是否存在
mysql> select case when max(emp_no) is NULL then 0 else 1 end as boolean from t_order where emp_no=1000000 limit 1;
+---------+
| boolean |
+---------+
|       0 |
+---------+
1 row in set (0.00 sec)

mysql> select case when max(emp_no) is NULL then 0 else 1 end as boolean from t_order where emp_no=10004 limit 1;
+---------+
| boolean |
+---------+
|       1 |
+---------+
1 row in set (0.00 sec)


--使用min max函数无论where条件如何，总有结果返回的特性：
mysql> select emp_no,first_name,now() s1 from emp3 where first_name='Sumali'
    -> union all
    -> select max(emp_no),max(first_name) first_name ,ifnull(max(now()),now()) s1 from emp3 where first_name='sumali';
+--------+------------+---------------------+
| emp_no | first_name | s1                  |
+--------+------------+---------------------+
|  10156 | Sumali     | 2019-04-10 21:08:07 |
|  10462 | Sumali     | 2019-04-10 21:08:07 |
|  10761 | Sumali     | 2019-04-10 21:08:07 |
|  10929 | Sumali     | 2019-04-10 21:08:07 |
|  10989 | Sumali     | 2019-04-10 21:08:07 |
|  10989 | Sumali     | 2019-04-10 21:08:07 |
+--------+------------+---------------------+
6 rows in set (0.00 sec)

mysql> select emp_no,first_name,now() s1 from emp3 where first_name='Sumali2'
    -> union all
    -> select max(emp_no),max(first_name) first_name ,ifnull(max(now()),now()) s1 from emp3 where first_name='sumali2';
+--------+------------+---------------------+
| emp_no | first_name | s1                  |
+--------+------------+---------------------+
|   NULL | NULL       | 2019-04-10 21:08:20 |
+--------+------------+---------------------+
1 row in set (0.01 sec)

mysql> select emp_no,first_name,now() s1 from emp3 where first_name='Sumali'
    -> union 
    -> select max(emp_no),max(first_name) first_name ,ifnull(max(now()),now()) s1 from emp3 where first_name='sumali';
+--------+------------+---------------------+
| emp_no | first_name | s1                  |
+--------+------------+---------------------+
|  10156 | Sumali     | 2019-04-10 21:08:37 |
|  10462 | Sumali     | 2019-04-10 21:08:37 |
|  10761 | Sumali     | 2019-04-10 21:08:37 |
|  10929 | Sumali     | 2019-04-10 21:08:37 |
|  10989 | Sumali     | 2019-04-10 21:08:37 |
+--------+------------+---------------------+
5 rows in set (0.00 sec)

```



### SUM AVG

只能对数字 或者 能转换成数字类型的 字符计算，不包括NULL

```sql
mysql> select avg(emp_no),sum(emp_no)/count(*),sum(emp_no),count(*),count(emp_no) from t_order;
+-------------+----------------------+-------------+----------+---------------+
| avg(emp_no) | sum(emp_no)/count(*) | sum(emp_no) | count(*) | count(emp_no) |
+-------------+----------------------+-------------+----------+---------------+
|  34250.3333 |           30825.3000 |      308253 |       10 |             9 |
+-------------+----------------------+-------------+----------+---------------+

mysql> select avg(ifnull(emp_no,0)) from t_order;
+-----------------------+
| avg(ifnull(emp_no,0)) |
+-----------------------+
|            30825.3000 |
+-----------------------+
1 row in set (0.00 sec)

```



### HAVING

```sql
mysql> select dept_no,count(*) from t_group group by dept_no having count(*)=1 limit 2;
+---------+----------+
| dept_no | count(*) |
+---------+----------+
| d002    |        1 |
| d004    |        1 |
+---------+----------+
2 rows in set (0.30 sec)
--<=>等于<=>--
mysql> select a.* from (select dept_no,count(*) c from t_group group by dept_no ) a where a.c=1 limit 2;
+---------+---+
| dept_no | c |
+---------+---+
| d002    | 1 |
| d004    | 1 |
+---------+---+
2 rows in set (0.00 sec)

--错误案例
mysql> select 'abc' as hi from (select 1) c where hi='abc';
ERROR 1054 (42S22): Unknown column 'hi' in 'where clause'
说明： where条件当中的hi 指的是from 后面子查询中的hi 列，但是子查询中无hi列所以报错！

--如何改写
mysql> select 'abc' as hi from (select 1) c having hi='abc';
+-----+
| hi  |
+-----+
| abc |
+-----+
1 row in set (0.00 sec)
--可见having可以减少一层子查询的嵌套
--还可以改写成：
mysql> select * from (select 'abc' as hi from (select 1) c) a where a.hi = 'abc';
+-----+
| hi  |
+-----+
| abc |
+-----+
1 row in set (0.00 sec)
```



### SUBQUERY

按照子查询出现的位置可以分为 INLINE VIEW SUBQUERY、SCALA SUBQUERY、IN EXISTS使用的SUBQUERY

#### inline view SUBQUERY

用在from后面，用括号括起来，相当于一个表：

5.7以上版本，若子查询中不包含集合函数 union all 、union、limit等关键字，当心视图合并。

查看执行计划，保证该SUBQUERY第一个被执行或者不出现 temp file 、filesort 等执行计划。

​          

```sql
mysql> select count(*) from employees e join (select * from salaries where salary between 6000 and 66961) t on e.emp_no=t.emp_no;
+----------+
| count(*) |
+----------+
|  1773559 |
+----------+
1 row in set (7.55 sec)

--视图合并：
mysql> desc select count(*) from employees e join (select * from salaries where salary between 6000 and 66961) t on e.emp_no=t.emp_no;
+----+-------------+----------+------------+--------+----------------+---------+---------+---------------------------+---------+----------+-------------+
| id | select_type | table    | partitions | type   | possible_keys  | key     | key_len | ref                       | rows    | filtered | Extra       |
+----+-------------+----------+------------+--------+----------------+---------+---------+---------------------------+---------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | ALL    | PRIMARY,emp_no | NULL    | NULL    | NULL                      | 2727280 |    11.11 | Using where |
|  1 | SIMPLE      | e        | NULL       | eq_ref | PRIMARY        | PRIMARY | 4       | employees.salaries.emp_no |       1 |   100.00 | Using index |
+----+-------------+----------+------------+--------+----------------+---------+---------+---------------------------+---------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

mysql> desc select count(*) from employees e join (select * from salaries where salary between 6000 and 66961 limit 20000000) t on e.emp_no=t.emp_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows    | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |  303000 |   100.00 | NULL        |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | t.emp_no |       1 |   100.00 | Using index |
|  2 | DERIVED     | salaries   | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 2727280 |    11.11 | Using where |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

mysql> select count(*) from employees e join (select * from salaries where salary between 6000 and 66961 limit 20000000) t on e.emp_no=t.emp_no;
+----------+
| count(*) |
+----------+
|  1773559 |
+----------+
1 row in set (4.24 sec)
--去除视图合并后效果明显
mysql> select count(*) from employees e straight_join (select * from salaries where salary between 6000 and 66961 limit 20000000) t on e.emp_no=t.emp_no;
+----------+
| count(*) |
+----------+
|  1773559 |
+----------+
1 row in set (9.15 sec)

mysql> desc select count(*) from employees e straight_join (select * from salaries where salary between 6000 and 66961 limit 20000000) t on e.emp_no=t.emp_no;
+----+-------------+------------+------------+-------+---------------+-------------+---------+--------------------+---------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key         | key_len | ref                | rows    | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+-------------+---------+--------------------+---------+----------+-------------+
|  1 | PRIMARY     | e          | NULL       | index | PRIMARY       | PRIMARY     | 4       | NULL               |  299202 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 4       | employees.e.emp_no |      10 |   100.00 | NULL        |
|  2 | DERIVED     | salaries   | NULL       | ALL   | NULL          | NULL        | NULL    | NULL               | 2727280 |    11.11 | Using where |
+----+-------------+------------+------------+-------+---------------+-------------+---------+--------------------+---------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)
```

 

#### SCALA SUBQUERY

```sql
这种子查询（标量子查询）相当于函数，要求子查询返回1条或者0条数据：
这种子查询一边运行，一边检查；所以不要在返回多行的查询中使用，而且子查询中where条件的列，必须要有很好的索引！

往往可以和left join互相替换（左右表的连接条件满足N：1）：
但是在mysql优化中，往往把left join改成标量子查询的情况较多！（比如含有limit 时）
```



```sql
mysql> select count(emp_no) from dept_emp ;
+---------------+
| count(emp_no) |
+---------------+
|        331603 |
+---------------+
1 row in set (0.04 sec)


mysql> select count(distinct(emp_no)) from dept_emp ;
+-------------------------+
| count(distinct(emp_no)) |
+-------------------------+
|                  300024 |
+-------------------------+
emp_no列当中有重复的行：

但是运行结果如下，显示边运行，一边判断，有可能在多行查询中，可能在SQL运行的中途报错，外层的取值放在内层中去循环判断：
mysql> select t1.emp_no,(select e.from_date from dept_emp e where e.emp_no=t1.emp_no) from_date from t1 limit 100;
ERROR 1242 (21000): Subquery returns more than 1 row

mysql> select t1.emp_no,(select e.from_date from dept_emp e where e.emp_no=t1.emp_no) from_date from t1 limit 10;
+--------+------------+
| emp_no | from_date  |
+--------+------------+
| 110022 | 1985-01-01 |
| 110085 | 1985-01-01 |
| 110183 | 1985-01-01 |
| 110303 | 1985-01-01 |
| 110511 | 1985-01-01 |
| 110725 | 1985-01-01 |
| 111035 | 1985-01-01 |
| 111400 | 1985-01-01 |
| 111692 | 1985-01-01 |
| 110114 | 1985-01-14 |
+--------+------------+
10 rows in set (0.00 sec)

--标量子查询where没有索引，无法添加索引的优化办法：
mysql> select t1.emp_no,(select e.first_name from employees e IGNORE INDEX(PRIMARY) where e.emp_no=t1.emp_no) first_name from t1 limit 100;
100 rows in set (5.87 sec)

mysql> select t1.emp_no,(select e.first_name from employees e IGNORE INDEX(PRIMARY) where e.emp_no=t1.emp_no limit 1) first_name from t1 limit 100;
100 rows in set (1.94 sec)
--添加limit 1效果明显！表示匹配到符合条件的值就不在内层循环查下去了 ，因此速度提升
```



##### 标量子查询和left join互相替换

```sql
标量子查询和left join互相替换；
mysql> desc select t1.emp_no,(select e.first_name from employees e where e.emp_no=t1.emp_no limit 1) first_name from t1 limit 100;
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2681514 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL        |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)

mysql> desc select t1.emp_no,e.first_name from t1 left join employees e on e.emp_no=t1.emp_no limit 100;
+----+-------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2681514 |   100.00 | Using index |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

--left join 改写成 标量子查询
mysql> select t1.emp_no,first_name from t1 left join (select e.emp_no,max(e.first_name) first_name from employees e group by e.emp_no) e on t1.emp_no=e.emp_no limit 6;
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 110022 | Margareta  |
| 110085 | Ebru       |
| 110183 | Shirish    |
| 110303 | Krassimir  |
| 110511 | DeForest   |
| 110725 | Peternela  |
+--------+------------+
6 rows in set (0.81 sec)

mysql> select t1.emp_no,(select max(e.first_name) from employees e where t1.emp_no=e.emp_no group by e.emp_no) first_name from t1 limit 6;
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 110022 | Margareta  |
| 110085 | Ebru       |
| 110183 | Shirish    |
| 110303 | Krassimir  |
| 110511 | DeForest   |
| 110725 | Peternela  |
+--------+------------+
6 rows in set (0.00 sec)

mysql> desc select t1.emp_no,first_name from t1 left join (select e.emp_no,max(e.first_name) first_name from employees e group by e.emp_no) e on t1.emp_no=e.emp_no limit 6;
+----+-------------+------------+------------+-------+---------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY     | t1         | NULL       | index | NULL          | ix_from_date | 3       | NULL                | 2681514 |   100.00 | Using index |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0>  | 4       | employees.t1.emp_no |      10 |   100.00 | NULL        |
|  2 | DERIVED     | e          | NULL       | index | PRIMARY       | PRIMARY      | 4       | NULL                |  299202 |   100.00 | NULL        |
+----+-------------+------------+------------+-------+---------------+--------------+---------+---------------------+---------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)

mysql> desc select t1.emp_no,(select max(e.first_name) from employees e where t1.emp_no=e.emp_no group by e.emp_no) first_name from t1 limit 6;    
| id | select_type        | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2681514 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | Using where |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)


8.0版本可以使用with子句：
mysql> with s as (select t1.emp_no from t1 limit 5)
    -> select t1.emp_no ,first_name from s t1 
    -> left join (select e.emp_no,max(e.first_name) first_name from s straight_join employees e on s.emp_no=e.emp_no) e
    -> on t1.emp_no=e.emp_no limit 5;
```



##### Scala SUBQUERY 可以进行分页查询

```sql
mysql> select  t1.emp_no,first_name from t1 left join (select e.emp_no,max(e.first_name) first_name from employees e group by e.emp_no) e 
    -> on t1.emp_no=e.emp_no limit 100000,5;
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 431298 | Jaroslava  |
| 433706 | Hatem      |
| 436996 | Weiru      |
| 438054 | Sariel     |
| 441872 | Mantis     |
+--------+------------+
5 rows in set (2.95 sec)

mysql> select t1.emp_no,(select max(first_name) from employees e where e.emp_no=t1.emp_no group by e.emp_no) first_name from t1 limit 100000,5;
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 431298 | Jaroslava  |
| 433706 | Hatem      |
| 436996 | Weiru      |
| 438054 | Sariel     |
| 441872 | Mantis     |
+--------+------------+
5 rows in set (0.01 sec)
--分页打印效果拔群
```



### In  exists

#### 在mysql当中 IN可能被打开成 join

```SQL
案例1：
mysql> desc select t1.emp_no from t1 where t1.emp_no in ( select e.emp_no from employees e where e.emp_no between 10001 and 12000) limit 5;
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys  | key     | key_len | ref                | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+--------------------------+
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY        | PRIMARY | 4       | NULL               | 2000 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | t1    | NULL       | ref   | PRIMARY,emp_no | emp_no  | 4       | employees.e.emp_no |    9 |   100.00 | Using index              |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> show warnings\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t1`.`emp_no` AS `emp_no` from `employees`.`employees` `e` join `employees`.`t1` where ((`employees`.`t1`.`emp_no` = `employees`.`e`.`emp_no`) and (`employees`.`e`.`emp_no` between 10001 and 12000)) limit 5
1 row in set (0.00 sec)

ERROR: 
No query specified
案例2：
mysql> desc select t1.emp_no from t1 where t1.emp_no in ( select e.emp_no from dept_emp e where e.emp_no between 10001 and 12000) limit 5;
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+-------------------------------------+
| id | select_type | table | partitions | type  | possible_keys  | key     | key_len | ref                | rows | filtered | Extra                               |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+-------------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY,emp_no | PRIMARY | 4       | NULL               | 2209 |   100.00 | Using where; Using index; LooseScan |
|  1 | SIMPLE      | t1    | NULL       | ref   | PRIMARY,emp_no | emp_no  | 4       | employees.e.emp_no |    9 |   100.00 | Using index                         |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+-------------------------------------+
2 rows in set, 1 warning (0.05 sec)

mysql> show warnings\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t1`.`emp_no` AS `emp_no` from `employees`.`t1` semi join (`employees`.`dept_emp` `e`) where ((`employees`.`t1`.`emp_no` = `employees`.`e`.`emp_no`) and (`employees`.`e`.`emp_no` between 10001 and 12000)) limit 5
1 row in set (0.00 sec)

--案例1和2 优化器的改写结果不同，因为1中in后面子查询的where条件针对的是主键，2中不是，可能会有重复值，需要利用semi join来去重！
```



#### IN 子句当中不支持limit的解决方法：多套一层子查询 或者改成join

```SQL
mysql> select t1.emp_no from t1 where t1.emp_no in ( select e.emp_no from employees e where e.emp_no between 10001 and 12000 limit 1000) limit 5;
ERROR 1235 (42000): This version of MySQL doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery

mysql> select t1.emp_no from t1 where t1.emp_no in (select s.* from ( select e.emp_no from employees e where e.emp_no between 10001 and 12000 limit 1000) s) limit 5;
--或者
mysql> select t1.emp_no from t1 join(
    -> select distinct a.emp_no from(select e.emp_no from employees e where e.emp_no between 10001 and 12000 limit 1000) a
    -> ) E on t1.emp_no = E.emp_no limit 5;
```

​          

#### exists 不能被优化器自动改写成join

注意，Mariadb可以打开exist to in的参数使得两者等价！

exists子句只能和标量子查询一样接受参数！  exists主要是针对表的连接条件是N:N或者1：N的时候，或者join之后结果集变的很大的情况！

```SQL
mysql> select t1.emp_no from t1 where exists (select e.emp_no from employees e where e.emp_no between 10001 and 12000 and e.emp_no=t1.emp_no) limit 5;
+--------+
| emp_no |
+--------+
|  10550 |
|  10583 |
|  10195 |
|  10009 |
|  10137 |
+--------+
5 rows in set (0.01 sec)

mysql> desc  select t1.emp_no from t1 where exists (select e.emp_no from employees e where e.emp_no between 10001 and 12000 and e.emp_no=t1.emp_no) limit 5;
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+--------------------------+
| id | select_type        | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra                    |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+--------------------------+
|  1 | PRIMARY            | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2681514 |   100.00 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | Using where; Using index |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+--------------------------+
2 rows in set, 2 warnings (0.00 sec)

mysql> show warnings\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1276
Message: Field or reference 'employees.t1.emp_no' of SELECT #2 was resolved in SELECT #1
*************************** 2. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t1`.`emp_no` AS `emp_no` from `employees`.`t1` where exists(/* select#2 */ select `employees`.`e`.`emp_no` from `employees`.`employees` `e` where ((`employees`.`e`.`emp_no` between 10001 and 12000) and (`employees`.`e`.`emp_no` = `employees`.`t1`.`emp_no`))) limit 5
2 rows in set (0.00 sec)
--可见exists没有被打开
ERROR: 
No query specified

mysql> select t1.emp_no ,(select e.first_name from employees e where e.emp_no=t1.emp_no) first_name from t1 limit 5;
+--------+------------+
| emp_no | first_name |
+--------+------------+
| 110022 | Margareta  |
| 110085 | Ebru       |
| 110183 | Shirish    |
| 110303 | Krassimir  |
| 110511 | DeForest   |
+--------+------------+
5 rows in set (0.00 sec)

mysql> desc select t1.emp_no ,(select e.first_name from employees e where e.emp_no=t1.emp_no) first_name from t1 limit 5;
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key          | key_len | ref                 | rows    | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | index  | NULL          | ix_from_date | 3       | NULL                | 2681514 |   100.00 | Using index |
|  2 | DEPENDENT SUBQUERY | e     | NULL       | eq_ref | PRIMARY       | PRIMARY      | 4       | employees.t1.emp_no |       1 |   100.00 | NULL        |
+----+--------------------+-------+------------+--------+---------------+--------------+---------+---------------------+---------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)

--可见，exists和标量子查询的执行计划类似
```



### 列转行  行转列

列转行 比较常见!

#### 行转列

```sql 
mysql> select * from t_group where dept_no='d008';
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
+--------+---------+------------+------------+

mysql> select min(case when emp_no=46554 then emp_no end) emp_no46554,
    -> min(case when emp_no=46554 then from_date end) from_date46554,
    -> min(case when emp_no=46554 then to_date end) to_date46554,
    -> min(case when emp_no=48317 then emp_no end) emp_no48317,
    -> min(case when emp_no=48317 then from_date end) from_date48317,
    -> min(case when emp_no=48317 then to_date end) to_date48317
    -> from t_group where dept_no='d008'
    -> group by dept_no;
+-------------+----------------+--------------+-------------+----------------+--------------+
| emp_no46554 | from_date46554 | to_date46554 | emp_no48317 | from_date48317 | to_date48317 |
+-------------+----------------+--------------+-------------+----------------+--------------+
|       46554 | 1986-12-01     | 1992-05-27   |       48317 | 1986-12-01     | 1989-01-11   |
+-------------+----------------+--------------+-------------+----------------+--------------+
```



#### 列转行 

```sql
mysql> select * from  grade;
+-----------+----------+-------+
| studyCode | subjectS | score |
+-----------+----------+-------+
| 001       | 数学     |   120 |
| 002       | 数学     |   130 |
| 003       | 数学     |   125 |
| 001       | 英语     |   130 |
| 002       | 英语     |   140 |
| 003       | 英语     |   135 |
| 001       | 国学     |   110 |
| 002       | 国学     |   136 |
| 003       | 国学     |   145 |
+-----------+----------+-------+
9 rows in set (0.00 sec)

mysql> SELECT  
    -> studyCode 学号,
    -> SUM(IF(subjectS = '国学',score,0)) 国学,
    -> SUM(IF(subjectS = '数学',score,0)) 数学,
    -> SUM(IF(subjectS = '英语',score,0)) 英语
    -> FROM grade
    -> GROUP BY studyCode;
+--------+--------+--------+--------+
| 学号   | 国学   | 数学   | 英语   |
+--------+--------+--------+--------+
| 001    |    110 |    120 |    130 |
| 002    |    136 |    130 |    140 |
| 003    |    145 |    125 |    135 |
+--------+--------+--------+--------+
3 rows in set (0.00 sec)

mysql> SELECT  
    -> studyCode 学号,
    -> SUM(CASE WHEN subjectS = '国学' THEN  score ELSE 0 END) 国学,
    -> SUM(CASE WHEN subjectS = '数学' THEN  score ELSE 0 END) 数学,
    -> SUM(CASE WHEN subjectS = '英语' THEN  score ELSE 0 END) 英语
    -> FROM grade
    -> GROUP BY studyCode;
+--------+--------+--------+--------+
| 学号   | 国学   | 数学   | 英语   |
+--------+--------+--------+--------+
| 001    |    110 |    120 |    130 |
| 002    |    136 |    130 |    140 |
| 003    |    145 |    125 |    135 |
+--------+--------+--------+--------+
3 rows in set (0.00 sec)
```



#### SIMPLE   QUERY

几项原则

#### 1 减少查询对象的数据页（db block）数量

```SQL
尽量避免使用*，使用准确的列名减少不必要的资源浪费!
表不能设计得太宽！

小技巧：
针对字段较多的表，要选择大部分字段查询，去掉少部分字段。利用：
mysql> select * from t_group limit 1\G;
*************************** 1. row ***************************
   emp_no: 22744
  dept_no: d006
from_date: 1986-12-01
  to_date: 9999-01-01
在把它放在 文本编辑器里面选择需要的字段 即可！
```



#### 2 查看是否使用了index

```sql
索引时SQL性能调优的重要手段，以下情况不能使用索引：
a. 对索引列进行加工
b. 索引列发生数据类型转换
c. 使用不等号
d. like %a
```



#### 3 用index代替sort

```
排序的副作用是在内存中file sort，还会产生temp file
利用index有顺序来代替sort
```



#### 4 hint是最后手段

#### 5 减少or的使用，不如使用union

or条件必须加括号



#### 6 单纯的验证存在与否的时候 添加limit 1

```sql
实验：
--可见limit 1对结果无影响
mysql> select t.*,(select count(0) from dept_emp e where e.emp_no=t.emp_no limit 1) s from t_group t;
+--------+---------+------------+------------+------+
| emp_no | dept_no | from_date  | to_date    | s    |
+--------+---------+------------+------------+------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |    1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |    1 |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |    0 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |    1 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |    1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |    1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |    2 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |    1 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |    1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |    1 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |    1 |
+--------+---------+------------+------------+------+
11 rows in set (0.11 sec)

mysql> select t.*,(select count(0) from dept_emp e where e.emp_no=t.emp_no ) s from t_group t;
+--------+---------+------------+------------+------+
| emp_no | dept_no | from_date  | to_date    | s    |
+--------+---------+------------+------------+------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |    1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |    1 |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |    0 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |    1 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |    1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |    1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |    2 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |    1 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |    1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |    1 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |    1 |
+--------+---------+------------+------------+------+
11 rows in set (0.00 sec)
而且limit 1可以用于解决标量子查询中返回结果超过一行的报错：
mysql> select t.*,(select e.emp_no from dept_emp e where e.emp_no=t.emp_no ) s from t_group t;
ERROR 1242 (21000): Subquery returns more than 1 row
mysql> select t.*,(select e.emp_no from dept_emp e where e.emp_no=t.emp_no limit 1) s from t_group t;
+--------+---------+------------+------------+-------+
| emp_no | dept_no | from_date  | to_date    | s     |
+--------+---------+------------+------------+-------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | 22744 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | 24007 |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |  NULL |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | 31112 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | 40983 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | 46554 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | 48317 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | 49667 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | 50449 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | 10004 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 | 50449 |
+--------+---------+------------+------------+-------+
11 rows in set (0.00 sec)
```



#### 7 减少没必要的distinct

```sql
mysql> select distinct (t.dept_no),emp_no from t_group t;
+---------+--------+
| dept_no | emp_no |
+---------+--------+
| d006    |  22744 |
| d005    |  24007 |
| d005    |   3097 |
| d002    |  31112 |
| d005    |  40983 |
| d008    |  46554 |
| d008    |  48317 |
| d007    |  49667 |
| d005    |  50449 |
| d004    |  10004 |
+---------+--------+
10 rows in set (0.00 sec)

mysql> select distinct (t.dept_no) from t_group t;
+---------+
| dept_no |
+---------+
| d006    |
| d005    |
| d002    |
| d008    |
| d007    |
| d004    |
+---------+
6 rows in set (0.00 sec)
--可见去重不会只对单个字段去重！
mysql> select distinct t.dept_no,emp_no from t_group t;
+---------+--------+
| dept_no | emp_no |
+---------+--------+
| d006    |  22744 |
| d005    |  24007 |
| d005    |   3097 |
| d002    |  31112 |
| d005    |  40983 |
| d008    |  46554 |
| d008    |  48317 |
| d007    |  49667 |
| d005    |  50449 |
| d004    |  10004 |
+---------+--------+
10 rows in set (0.00 sec)
```





### complex query

几个原则

#### 1  一般来说in 比exists更有利  

   对于数据库表优化器而言，in更容易变成join

   exists只能变成标量子查询

​          

#### 2 尽量避免使用union，使用union all，避免union的排序

#### 3 避免笛卡尔积

#### 4 在outer join的时候

注意on连接条件以及where的条件对join结果的影响

```SQL
mysql> select a.*,d.* from t_group a left join departments d on a.dept_no =d.dept_no where dept_name='Finance';
+--------+---------+------------+------------+---------+-----------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name |
+--------+---------+------------+------------+---------+-----------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance   |
+--------+---------+------------+------------+---------+-----------+
1 row in set (0.00 sec)

mysql> select a.*,d.* from t_group a left join departments d on a.dept_no =d.dept_no and dept_name='Finance';
+--------+---------+------------+------------+---------+-----------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name |
+--------+---------+------------+------------+---------+-----------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|   3097 | d005    | 1986-12-01 | 2017-03-29 | NULL    | NULL      |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance   |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | NULL    | NULL      |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | NULL    | NULL      |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | NULL    | NULL      |
|  50449 | d005    | 1987-12-01 | 9999-01-01 | NULL    | NULL      |
+--------+---------+------------+------------+---------+-----------+
11 rows in set (0.28 sec)

mysql> desc select a.*,d.* from t_group a left join departments d on a.dept_no =d.dept_no where dept_name='Finance';
+----+-------------+-------+------------+-------+-------------------+-----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys     | key       | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-------------------+-----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | d     | NULL       | const | PRIMARY,dept_name | dept_name | 122     | const |    1 |   100.00 | Using index |
|  1 | SIMPLE      | a     | NULL       | ALL   | NULL              | NULL      | NULL    | NULL  |   10 |    10.00 | Using where |
+----+-------------+-------+------------+-------+-------------------+-----------+---------+-------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
--可以看到执行计划当中where条件使得join失效。


mysql> desc select a.*,d.* from t_group a left join departments d on a.dept_no =d.dept_no and dept_name='Finance';
+----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys     | key     | key_len | ref                 | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+------+----------+-------------+
|  1 | SIMPLE      | a     | NULL       | ALL    | NULL              | NULL    | NULL    | NULL                |   10 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | eq_ref | PRIMARY,dept_name | PRIMARY | 12      | employees.a.dept_no |    1 |    11.11 | Using where |
+----+-------------+-------+------------+--------+-------------------+---------+---------+---------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

```

  

#### 6 case when的特点和陷阱

   case ... when ...when ：when之间的判断条件不能有交集，否则，第二个判断条件不会执行



#### 7 group by  ... with rollup

```sql 
mysql> select (case when dept_no is NULL then '合计' else dept_no end) dept_no,c1,c2 from ( select dept_no,count(*) c1,
       sum(case when to_date='9999-01-01' then 1 else 0 end) c2  from t_group group by dept_no with rollup) a;
+---------+----+------+
| dept_no | c1 | c2   |
+---------+----+------+
| d002    |  1 |    0 |
| d004    |  1 |    1 |
| d005    |  5 |    4 |
| d006    |  1 |    1 |
| d007    |  1 |    1 |
| d008    |  2 |    0 |
| 合计    | 11 |    7 |
+---------+----+------+
7 rows in set (0.00 sec)

```







## 案例

### 标量子查询中含有 order by ...limit，使用聚合函数max   min  concat优化：

```sql 
标量子查询：
(select ec.input_name from ec where ec.user_id=b.user_id order by id limit 1) user_name1

优化后：
(select substring(min(concat(lpad(ec.id,10,'0'),ec.input_name)),11) from ec where ec.user_id=b.user_id) as user_name1
```



### group by优化：减少group by子句中函数运行的次数

```
select boxname,
(case when regcat is NULL then '' else regcat end) as regcat,
(case when re_id is NULL then '' else re_id end) as re_id,
(case when par_id is NULL then '' else par_id end) as par_id,
count (*) as his_count,
min(oid) as min_oid,
max(oid) as max_oid,
max(issutime) as max_issutime,
min(issutime) as min_issutime
from tablecount
group by 
boxname,
(case when regcat is NULL then '' else regcat end),
(case when re_id is NULL then '' else re_id end),
(case when par_id is NULL then '' else par_id end);


优化的思想：减少group by中case when条件子句的运行次数，减少对全表扫描的次数
select boxname,
(case when regcat is NULL then '' else regcat end) as regcat,
(case when re_id is NULL then '' else re_id end) as re_id,
(case when par_id is NULL then '' else par_id end) as par_id,
count (*) as his_count,
min(oid) as min_oid,
max(oid) as max_oid,
max(issutime) as max_issutime,
min(issutime) as min_issutime
from (
select boxname,regcat,re_id,par_id,
count (*) as his_count,
min(oid) as min_oid,
max(oid) as max_oid,
max(issutime) as max_issutime,
min(issutime) as min_issutime
from tablecount
group by 
boxname,
regcat,
re_id,
par_id
) a;
进一步优化，对group by子句后面的列创建联合索引！
```



### 延迟join的优化，利用延迟join，减少回表的次数

```SQL
案例：

延迟join的优化，利用延迟join，减少回表的次数：

select history.id as history,
assign.operator_id,
history.p_first_in,
assign.create_at,
assign.destroy_at,
history.current_stage,
assign.result,
....
from T_Collection_Case_History history
INNER join T_Collection_Case cases ON cases.id=history.case_id,
INNER JOIN T_Collection_Case_Assign assign ON assign.id=history.case_assign_id
where history.case_assign_id >= 155965 AND history.process_type IN (1,3) AND 
history.id > 594972 order by history.id ASC LIMIT 2000;
说明：提前join产生大量中间结果集，根据where条件索引回表，在如此多的结果集中读取出数据导致SQL执行效率不高。
可以考虑减少中间结果集并减少回表的次数，where条件（where条件只是对history表的字段）并且order by ...limit 2000提前有效减少join的中间结果，
并且用exists改写join连接条件：



初步优化：
select history.id as history,
assign.operator_id,
history.p_first_in,
assign.create_at,
assign.destroy_at,
history.current_stage,
assign.result,
....
from (
select history.XXXX  ---XXXX指的是外查询当中history相关的字段+join连接条件相关的字段
....
from T_Collection_Case_History history
where history.case_assign_id >= 155965 AND history.process_type IN (1,3) AND 
history.id > 594972
and exists (select 1 from T_Collection_Case cases where cases.id=history.case_id)
and exists (select 1 from T_Collection_Case_Assign where assign.id=history.case_assign_id) ---利用exists外循环满足即停止的特性
order by history.id ASC LIMIT 2000
) history  straight_join ----使得子查询作为驱动表和他表join，回表数据2000行
T_Collection_Case cases ON cases.id=history.case_id
INNER JOIN T_Collection_Case_Assign assign ON assign.id=history.case_assign_id;



最终优化：
select history.id as history,
assign.operator_id,
history.p_first_in,
assign.create_at,
assign.destroy_at,
history.current_stage,
assign.result,
....
from (
select history.id,history.case_id,history.case_assign_id
from T_Collection_Case_History history
where history.case_assign_id >= 155965 AND history.process_type IN (1,3) AND history.id > 594972
and exists (select 1 from T_Collection_Case cases where cases.id=history.case_id)
and exists (select 1 from T_Collection_Case_Assign where assign.id=history.case_assign_id)
order by history.id ASC LIMIT 2000  ----因为order by id是orcer by的条件，所以创建联合索引，id为前导列，减少file sort
) history1 straight_join T_Collection_Case cases ON cases.id=history.case_id
INNER JOIN T_Collection_Case_Assign assign ON assign.id=history.case_assign_id
inner join T_Collection_Case_History history on history.id=history1.id;

alter table T_Collection_Case_History add index idx_try3(id ,process_type,case_assign_id,case_id);
```

