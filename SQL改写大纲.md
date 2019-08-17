[TOC]

# SQL改写



## 1 从集合的观点看SQL

把join归为一部分，看做一个整体:

join之后经过过滤，整体行数较之前是变小的

而left join作为一部分：

left join之后整体的行数是不会有变化的（M:1）

```SQL
mysql> select count(*) from employees e left join (select distinct emp_no from salaries ) s on s.emp_no=e.emp_no;
+----------+
| count(*) |
+----------+
|   300024 |
+----------+
1 row in set (1.14 sec)

mysql> select count(*) from employees ;
+----------+
| count(*) |
+----------+
|   300024 |
+----------+
1 row in set (0.04 sec)
--所以count 函数计算left join右边的表是完全多余的
```

标量子查询看做一部分：

子查询的关联字段上必须要有索引

含有distinct的子查询，可以把distinct 去掉，添加limit 1

同一个表多次使用标量子查询，考虑使用concat，min/max优化

```sql
mysql> select t1.emp_no,(select e.first_name from employees e ignore index(primary) where e.emp_no=t1.emp_no limit 1) first_name,
    -> (select e.last_name from employees e ignore index(primary) where e.emp_no=t1.emp_no limit 1) last_name,
    -> (select e.hire_date from employees e ignore index(primary) where e.emp_no=t1.emp_no limit 1) hire_date
    -> from t1 limit 100;
    要运行6s 子查询个数*外层循环次数
    改写为：
mysql> select a.emp_no,substring_index(a.s1,'|',1) first_name,
    -> substring_index(substring_index(a.s1,'|',2),'|',2) last_name,
    -> substring_index(a.s1,'|',-1) hire_date
    -> from (
    -> select t1.emp_no,
    -> (select concat(e.first_name,'|',e.last_name,'|',e.hire_date) s1 from employees e IGNORE INDEX(primary)
    -> where e.emp_no=t1.emp_no limit 1) s1 
    -> from t1 limit 100) a; 运行3s
    
--标量子查询在最外层使用，利用limit减少循环次数
--如果外层数量太大 考虑改写成left join
```





## 2 去掉非必要的order by

### 使用适当的索引去掉务必要的order by

order by后面的字段上有索引，执行计划里面就不会有排序

```sql
mysql> desc select * from employees e,salaries s where s.emp_no=e.emp_no and e.emp_no between 10002 and 10020 order by s.emp_no;
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+----------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys  | key     | key_len | ref                | rows | filtered | Extra                                        |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | e     | NULL       | range | PRIMARY        | PRIMARY | 4       | NULL               |   19 |   100.00 | Using where; Using temporary; Using filesort |
|  1 | SIMPLE      | s     | NULL       | ref   | PRIMARY,emp_no | PRIMARY | 4       | employees.e.emp_no |    9 |   100.00 | NULL                                         |
+----+-------------+-------+------------+-------+----------------+---------+---------+--------------------+------+----------+----------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
出现Using temporary; Using filesort说明两个表join之后把结果放进临时表空间之后再进行排序，是比较差的执行计划
把order by后面改写为e.emp_no
```

### 子查询的order by 绝大部分，可以去掉

## 3 去掉join outer join

​    

## 4 人为进行子查询改写

子查询生成auto_key时，若结果集较小，则性能良好，结果集大，把子查询作为驱动表有改善！

```sql
mysql> desc select count(*) from employees d straight_join 
    -> (select emp_no from salaries group by emp_no) s on d.emp_no=s.emp_no;
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+--------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys  | key         | key_len | ref                | rows   | filtered | Extra                    |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+--------+----------+--------------------------+
|  1 | PRIMARY     | d          | NULL       | index | PRIMARY        | PRIMARY     | 4       | NULL               | 299202 |   100.00 | Using index              |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>    | <auto_key0> | 4       | employees.d.emp_no |     10 |   100.00 | Using index              |
|  2 | DERIVED     | salaries   | NULL       | range | PRIMARY,emp_no | emp_no      | 4       | NULL               | 289373 |   100.00 | Using index for group-by |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+--------+----------+--------------------------+
3 rows in set, 1 warning (0.00 sec)

mysql> select count(*) from employees d straight_join  (select emp_no from salaries group by emp_no) s on d.emp_no=s.emp_no;
+----------+
| count(*) |
+----------+
|   300024 |
+----------+
1 row in set (1.21 sec)

mysql> select count(*) from employees d straight_join  (select emp_no from salaries where emp_no between 10001 and 10010 group by emp_no) s on d.emp_no=s.emp_no;
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.09 sec)
```



## 5 去掉group by

```sql
案例：子查询中有group by无法打开视图，导致外层where条件无法传入：
mysql> select * from employees e inner join (select emp_no,count(*) from salaries group by emp_no) s 
    -> on s.emp_no=e.emp_no
    -> where e.emp_no between 10001 and 10010;
+--------+------------+------------+-----------+--------+------------+--------+----------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | emp_no | count(*) |
+--------+------------+------------+-----------+--------+------------+--------+----------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |  10001 |       17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |  10002 |        6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |  10003 |        7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |  10004 |       16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |  10005 |       13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |  10006 |       12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |  10007 |       14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |  10008 |        3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |  10009 |       18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |  10010 |        6 |
+--------+------------+------------+-----------+--------+------------+--------+----------+
10 rows in set (1.51 sec)
优化思路1：使用函数
delimiter $$
create function fo1 (in_emp int) returns int
DETERMINISTIC
begin
declare a1 int;
select  count(*) into a1 from salaries where emp_no = in_emp group by emp_no;
return a1;
end $$
delimiter ;

mysql> select e.* ,fo1(e.emp_no) from employees e where e.emp_no between 10001 and 10010 and fo1(e.emp_no);
+--------+------------+------------+-----------+--------+------------+---------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | fo1(e.emp_no) |
+--------+------------+------------+-----------+--------+------------+---------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |            17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |             6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |             7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |            16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |            13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |            12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |            14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |             3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |            18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |             6 |
+--------+------------+------------+-----------+--------+------------+---------------+

mysql> select e.* ,fo1(e.emp_no) from employees e where e.emp_no between 10001 and 10010 ;
+--------+------------+------------+-----------+--------+------------+---------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | fo1(e.emp_no) |
+--------+------------+------------+-----------+--------+------------+---------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |            17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |             6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |             7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |            16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |            13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |            12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |            14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |             3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |            18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |             6 |
+--------+------------+------------+-----------+--------+------------+---------------+

思路2：将外面的值传入子查询 叫filter push down

mysql> select * from employees e inner join (select emp_no,count(*) from salaries where emp_no between 10001 and 10010 group by emp_no) s
    -> on s.emp_no=e.emp_no;
+--------+------------+------------+-----------+--------+------------+--------+----------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | emp_no | count(*) |
+--------+------------+------------+-----------+--------+------------+--------+----------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |  10001 |       17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |  10002 |        6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |  10003 |        7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |  10004 |       16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |  10005 |       13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |  10006 |       12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |  10007 |       14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |  10008 |        3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |  10009 |       18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |  10010 |        6 |
+--------+------------+------------+-----------+--------+------------+--------+----------+
10 rows in set (0.00 sec)
mysql> select * from employees e inner join (select e.emp_no,count(*) from salaries s inner join employees e on  s.emp_no=e.emp_no 
    -> where e.emp_no between 10001 and 10010 group by s.emp_no) s
    -> on s.emp_no=e.emp_no 
    -> where e.emp_no between 10001 and 10010;
+--------+------------+------------+-----------+--------+------------+--------+----------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | emp_no | count(*) |
+--------+------------+------------+-----------+--------+------------+--------+----------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |  10001 |       17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |  10002 |        6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |  10003 |        7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |  10004 |       16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |  10005 |       13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |  10006 |       12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |  10007 |       14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |  10008 |        3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |  10009 |       18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |  10010 |        6 |
+--------+------------+------------+-----------+--------+------------+--------+----------+
10 rows in set (0.00 sec)

案例2：利用ICP和创建不影响SQL结果的条件优化SQL
优化前：6s
select ...
from t force index(ix) where 
start_at between timestampadd(day,20,assign_at)
and timestampadd(day,29,assign_at)
and assign_at > date_sub(now(),interval 1 month)
group by operator_id,group_id;

思路：
由于assgin_at进行了timestampadd函数运算，导致assign_at上的索引无法使用，
5.6无法创建函数索引以及虚拟列
接着逐个检查where条件。发现assign_at > date_sub(now(),interval 1 month)的筛选结果不明显。
利用如下条件来提高筛选：
and start_at > timestampadd(day,20,date_sub(now(),interval 1 month))  大于assign_at的最小值，并不影响SQL结果。
利用好的筛选条件，使用ICP特性，索引中过滤数据，不回表

所以优化后：2.5s
select ...
from t force index(ix) where 
start_at between timestampadd(day,20,assign_at)
and timestampadd(day,29,assign_at)
and assign_at > date_sub(now(),interval 1 month)
and start_at > timestampadd(day,20,date_sub(now(),interval 1 month))
group by operator_id,group_id;
发现where条件中只和assign_at和start_at有关，创建联合索引：
create index ix on t1(start_at,assign_at)

--ps:timestampadd函数：
mysql> select now(),timestampadd(day,20,now()),date_sub(now(),interval 1 month);
+---------------------+----------------------------+----------------------------------+
| now()               | timestampadd(day,20,now()) | date_sub(now(),interval 1 month) |
+---------------------+----------------------------+----------------------------------+
| 2019-05-07 20:14:19 | 2019-05-27 20:14:19        | 2019-04-07 20:14:19              |
+---------------------+----------------------------+----------------------------------+
1 row in set (0.04 sec)


案例3：left join改写成subquery去掉group by：
mysql> select d.* ,a.* from t_group d left join (select emp_no,count(*) c from dept_emp group by emp_no) a on d.emp_no=a.emp_no;
+--------+---------+------------+------------+--------+------+
| emp_no | dept_no | from_date  | to_date    | emp_no | c    |
+--------+---------+------------+------------+--------+------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |  22744 |    1 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |  24007 |    1 |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |   NULL | NULL |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |  31112 |    1 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |  40983 |    1 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |  46554 |    1 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |  48317 |    2 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |  49667 |    1 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |  50449 |    1 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |  10004 |    1 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |  50449 |    1 |
+--------+---------+------------+------------+--------+------+
11 rows in set (0.46 sec)
mysql> select d.*,(select count(*) c from dept_emp a where d.emp_no=a.emp_no) c from t_group d;
+--------+---------+------------+------------+------+
| emp_no | dept_no | from_date  | to_date    | c    |
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
11 rows in set (0.29 sec)
```



## 6 order by 

select的列过多导致的性能分析（字段太多）

避免使用* 排序；

### 使用延迟join减少排序的字段：

select e.* from employees e order by e.emp_no desc limit 100;

select b.* from (select e.emp_no from employees e order by e.emp_no desc limit 100) e straight_join employees b on b.emp_no=e.emp_no;

```sql
案例：
select 多列cbo.* ... r.total_days
from cbo_w cbo
left outer join wf c on cbo.br_order_no=c.br_order_no
left outer join cr r on cbo.br_order_no=r.br_order_no
where cbo.br_state=8 and cbo.repay_state in ('3','4')
order by c.last_update_time,c.create_time,cbo.last_update_time desc limit 25;

改写为：

select 多列cbo.* ... (select r.total_days from cr r where cbo.br_order_no=r.br_order_no limit 1) total_days
from (select cbo.id 
from cbo_w cbo
left outer join wf c on cbo.br_order_no=c.br_order_no
where cbo.br_state=8 and cbo.repay_state in ('3','4')
order by c.last_update_time,c.create_time,cbo.last_update_time desc limit 25) cb 
straight_join cbo_w cbo on cb.id=cbo.id;
```





## 7 filter push down（谓词缩进）

```SQL
mysql> desc select * from employees e inner join (select emp_no,count(*) from salaries group by emp_no) s on s.emp_no=e.emp_no where e.emp_no between 10001 and 10010;
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+---------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys  | key         | key_len | ref                | rows    | filtered | Extra       |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+---------+----------+-------------+
|  1 | PRIMARY     | e          | NULL       | range | PRIMARY        | PRIMARY     | 4       | NULL               |      10 |   100.00 | Using where |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>    | <auto_key0> | 4       | employees.e.emp_no |      10 |   100.00 | NULL        |
|  2 | DERIVED     | salaries   | NULL       | index | PRIMARY,emp_no | emp_no      | 4       | NULL               | 2727280 |   100.00 | Using index |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+---------+----------+-------------+

mysql> select * from employees e inner join (select emp_no,count(*) from salaries group by emp_no) s on s.emp_no=e.emp_no where e.emp_no between 10001 and 10010;
+--------+------------+------------+-----------+--------+------------+--------+----------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | emp_no | count(*) |
+--------+------------+------------+-----------+--------+------------+--------+----------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |  10001 |       17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |  10002 |        6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |  10003 |        7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |  10004 |       16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |  10005 |       13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |  10006 |       12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |  10007 |       14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |  10008 |        3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |  10009 |       18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |  10010 |        6 |
+--------+------------+------------+-----------+--------+------------+--------+----------+
10 rows in set (0.69 sec)

谓词缩进通用方法：
mysql> desc select * from employees e inner join (select s.emp_no,count(*) from employees e inner join salaries s on s.emp_no=e.emp_no where e.emp_no between 10001 and 10010 group by s.emp_no) s on s.emp_no=e.emp_no where e.emp_no between 10001 and 10010;
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+------+----------+-----------------------------------------------------------+
| id | select_type | table      | partitions | type  | possible_keys  | key         | key_len | ref                | rows | filtered | Extra                                                     |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+------+----------+-----------------------------------------------------------+
|  1 | PRIMARY     | e          | NULL       | range | PRIMARY        | PRIMARY     | 4       | NULL               |   10 |   100.00 | Using where                                               |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>    | <auto_key0> | 4       | employees.e.emp_no |    9 |   100.00 | NULL                                                      |
|  2 | DERIVED     | e          | NULL       | range | PRIMARY        | PRIMARY     | 4       | NULL               |   10 |   100.00 | Using where; Using index; Using temporary; Using filesort |
|  2 | DERIVED     | s          | NULL       | ref   | PRIMARY,emp_no | PRIMARY     | 4       | employees.e.emp_no |    9 |   100.00 | Using index                                               |
+----+-------------+------------+------------+-------+----------------+-------------+---------+--------------------+------+----------+-----------------------------------------------------------+
4 rows in set, 1 warning (0.00 sec)

mysql> select * from employees e inner join (select s.emp_no,count(*) from employees e inner join salaries s on s.emp_no=e.emp_no where e.emp_no between 10001 and 10010 group by s.emp_no) s on s.emp_no=e.emp_no where e.emp_no between 10001 and 10010;
+--------+------------+------------+-----------+--------+------------+--------+----------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | emp_no | count(*) |
+--------+------------+------------+-----------+--------+------------+--------+----------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |  10001 |       17 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |  10002 |        6 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |  10003 |        7 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |  10004 |       16 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |  10005 |       13 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |  10006 |       12 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |  10007 |       14 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |  10008 |        3 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |  10009 |       18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |  10010 |        6 |
+--------+------------+------------+-----------+--------+------------+--------+----------+
10 rows in set (0.00 sec)


```



## 8 simple view merging

简单视图合并在如下情况下失效：

子查询中含有：distinct，union，group by，limit ，@，hint

## 9 complex view merging

子查询中含有：distinct，union，group by，limit ，@，hint，视图不能合并

## 10 给开发找数据

快速找出满足where条件的变量值

1）把where条件的值输入select列，where条件删除

2）把left join删除

3）group by，聚合函数删除

4）添加limit 1

```SQL
 原始SQL，需要找到满足条件不同的salary的值：
 select count(*) from employees d join (select emp_no from salaries where salary between 6000 and 66391 group by emp_no) t on d.emp_no=t.emp_no;
 按照如上思路修改SQL：
 mysql> select t.salary from employees d join (select emp_no ,salary from salaries) t on d.emp_no=t.emp_no limit 2;
+--------+
| salary |
+--------+
|  60117 |
|  62102 |
+--------+
2 rows in set (0.00 sec)
将取得的值带入原始SQL验证：
select count(*) from employees d join (select emp_no from salaries where salary between 60117 and 62102 group by emp_no) t on d.emp_no=t.emp_no; 
+----------+
| count(*) |
+----------+
|    93210 |
+----------+
1 row in set (0.57 sec)





```

 

## 11 使用@实现分组排序

```sql
mysql> select t.* ,if(@dept_no=t.dept_no ,@rn:=@rn+1,@rn:=1) as rn,@dept_no:=t.dept_no as calc_dept_no
    -> from (select *
    -> from t_group t order by t.dept_no,t.emp_no) t,(select @rn:=0 rn,@dept_no:='') b;
+--------+---------+------------+------------+------+--------------+
| emp_no | dept_no | from_date  | to_date    | rn   | calc_dept_no |
+--------+---------+------------+------------+------+--------------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 |    1 | d002         |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |    1 | d004         |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |    1 | d005         |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |    2 | d005         |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |    3 | d005         |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |    4 | d005         |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |    5 | d005         |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |    1 | d006         |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |    1 | d007         |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |    1 | d008         |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |    2 | d008         |
+--------+---------+------------+------------+------+--------------+
11 rows in set (0.06 sec)
```



## 案例1：

创建索引与filter列判断

查看执行计划中rows 太大，且filter很小，筛选合适的where条件创建索引。（选择选择率好的列创建联合索引）

## 案例2：

```SQL
select u.id,u.username,u.realname,u.ip
from t1 u left join
t2 c on u.id=c.user_id
where 
(u.realname='XXX'
or u.phone='XXX'
or u.ip='XXX'
or c.card_num='XXX')
and (u.id not in (select user_id from t3 where ralate_id='1000'));
--执行结果发现去掉条件c.card_num后速度飞快
select u.id,u.username,u.realname,u.ip
from t1 u
where 
(u.realname='XXX'
or u.phone='XXX'
or u.ip='XXX') 
and (u.id not in (select user_id from t3 where ralate_id='1000'))
union 
select u.* from (
select u.id,u.username,u.realname,u.ip
from t1 u
where 
(u.realname='XXX'
or u.phone='XXX'
or u.ip='XXX') 
and (u.id not in (select user_id from t3 where ralate_id='1000'))) u 
left join
t2 c on c.user_id=u.id 
where c.card_num='XXX';
```

