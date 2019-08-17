[TOC]

# join的实现与优化

## 为什么join

```sql
mysql> select e.emp_no,e.birth_date,d.dept_name from 
       employees e 
       join dept_emp de on e.emp_no=de.emp_no and to_date='9999-01-01' 
       join departments d on de.dept_no=d.dept_no where e.emp_no=10001;
+--------+------------+-------------+
| emp_no | birth_date | dept_name   |
+--------+------------+-------------+
|  10001 | 1953-09-02 | Development |
+--------+------------+-------------+
--通过中间表dept_emp连接employees和department：注意连接条件是n:1的关系，n:n会出现重复值
如：
mysql> select a.emp_no,a.from_date from t_group a
    -> join (select * from departments union all select * from  departments) d on a.dept_no=d.dept_no and d.dept_name='Finance';
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  31112 | 1986-12-01 |
|  31112 | 1986-12-01 |
+--------+------------+

--使用exists代替join
执行计划以dependent subquery执行，其运行次数与外循环次数相当，并且内部判断条件必须包含索引，才会有较好的性能。
mysql> select a.emp_no,a.from_date from t_group2 a where exists (select  d.dept_no from dept2 d where d.dept_name='Finance' and d.dept_no=a.dept_no);
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  31112 | 1986-12-01 |
+--------+------------+
1 row in set (0.00 sec)

mysql> desc  select a.emp_no,a.from_date from t_group2 a where exists (select  d.dept_no from dept2 d where d.dept_name='Finance' and d.dept_no=a.dept_no);
+----+--------------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | a     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | d     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   18 |     5.56 | Using where |
+----+--------------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)

--使用in代替join
mysql> select a.emp_no,a.from_date from t_group2 a where a.dept_no in (select d.dept_no from dept2 d where d.dept_name='Finance') ;
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  31112 | 1986-12-01 |
+--------+------------+
1 row in set (0.00 sec)

mysql> desc select a.emp_no,a.from_date from t_group2 a where a.dept_no in (select d.dept_no from dept2 d where d.dept_name='Finance') ;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                                             |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------------------------------------------+
|  1 | SIMPLE      | a     | NULL       | ALL  | ix_dept_no2   | NULL | NULL    | NULL |   10 |   100.00 | NULL                                                              |
|  1 | SIMPLE      | d     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   18 |     5.56 | Using where; FirstMatch(a); Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

--优化in思路，缩小结果集再join，是用小的结果集作为驱动表
mysql> select a.emp_no,a.from_date from
    -> (select distinct d.dept_no from dept2 d where d.dept_name='Finance') d join t_group2 a on a.dept_no=d.dept_no;
+--------+------------+
| emp_no | from_date  |
+--------+------------+
|  31112 | 1986-12-01 |
+--------+------------+
1 row in set (0.29 sec)

mysql> desc select a.emp_no,a.from_date from (select distinct d.dept_no from dept2 d where d.dept_name='Finance') d join t_group2 a on a.dept_no=d.dept_no;
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+------------------------------+
| id | select_type | table      | partitions | type | possible_keys | key         | key_len | ref       | rows | filtered | Extra                        |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |    2 |   100.00 | NULL                         |
|  1 | PRIMARY     | a          | NULL       | ref  | ix_dept_no2   | ix_dept_no2 | 12      | d.dept_no |    1 |   100.00 | NULL                         |
|  2 | DERIVED     | d          | NULL       | ALL  | NULL          | NULL        | NULL    | NULL      |   18 |    10.00 | Using where; Using temporary |
+----+-------------+------------+------------+------+---------------+-------------+---------+-----------+------+----------+------------------------------+
3 rows in set, 1 warning (0.00 sec)
```

### 1 范式

### 2 outer join

outer join的前提条件必须满足join之后，数据量不能有变化

数据量变多说明有重复值，说明被连接的表中有重复值。

```sql
mysql> select a.* ,d.* from t_group a left join departments d on a.dept_no=d.dept_no and dept_name='Finance';
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
11 rows in set (0.03 sec)

使用where条件，left join被转换了：
mysql> select a.* ,d.* from t_group a left join departments d on a.dept_no=d.dept_no where dept_name='Finance';
+--------+---------+------------+------------+---------+-----------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name |
+--------+---------+------------+------------+---------+-----------+
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance   |
+--------+---------+------------+------------+---------+-----------+
1 row in set (0.01 sec)

mysql> desc select a.* ,d.* from t_group a left join departments d on a.dept_no=d.dept_no where dept_name='Finance';
+----+-------------+-------+------------+-------+-------------------+-----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys     | key       | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-------------------+-----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | d     | NULL       | const | PRIMARY,dept_name | dept_name | 122     | const |    1 |   100.00 | Using index |
|  1 | SIMPLE      | a     | NULL       | ALL   | NULL              | NULL      | NULL    | NULL  |   10 |    10.00 | Using where |
+----+-------------+-------+------------+-------+-------------------+-----------+---------+-------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

mysql> show warnings\G;
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`a`.`emp_no` AS `emp_no`,`employees`.`a`.`dept_no` AS `dept_no`,`employees`.`a`.`from_date` AS `from_date`,`employees`.`a`.`to_date` AS `to_date`,'d002' AS `dept_no`,'Finance' AS `dept_name` from `employees`.`t_group` `a` join `employees`.`departments` `d` where ((`employees`.`a`.`dept_no` = 'd002') and ('Finance' = 'Finance'))
1 row in set (0.00 sec)
```



#### outer join的转换

outer join 可以改写成subquery

```sql
mysql> select a.* ,(select distinct d.dept_no from dept2 d where d.dept_no=a.dept_no) dept_no , (select distinct d.dept_name from dept2 d where d.dept_no=a.dept_no) dept_name from t_group3 a limit 10;
+--------+---------+------------+------------+---------+--------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name          |
+--------+---------+------------+------------+---------+--------------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | Quality Management |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | Development        |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | Finance            |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | Research           |
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL               |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | Sales              |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | Development        |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | Production         |
+--------+---------+------------+------------+---------+--------------------+
--如何改写 ，使得标量子查询只使用一次？
mysql> select c.emp_no,c.dept_no,c.from_date,c.to_date,substring_index(c.d,'_',1) dept_no, substring_index(c.d,'_',2) dept_name from
      ( select a.* ,(select concat(d.dept_no,'_',d.dept_name) from dept2 d where a.dept_no=d.dept_no limit 1) d from t_group3 a) c limit 10;
+--------+---------+------------+------------+---------+-------------------------+
| emp_no | dept_no | from_date  | to_date    | dept_no | dept_name               |
+--------+---------+------------+------------+---------+-------------------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | d006    | d006_Quality Management |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | d005    | d005_Development        |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | d005    | d005_Development        |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | d002    | d002_Finance            |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | d005    | d005_Development        |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | d008    | d008_Research           |
|  48317 | NULL    | 1986-12-01 | 1989-01-11 | NULL    | NULL                    |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | d007    | d007_Sales              |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | d005    | d005_Development        |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | d004    | d004_Production         |
+--------+---------+------------+------------+---------+-------------------------+
10 rows in set (0.01 sec)

案例1：
select 多个字段 from 多表连接 where (p.cateid='2726' or p.cateid in (select id from products where pid='2726' or rootid='2726')) 
and o.addtime >= unix_timestamp('2018-10-12 00:00:00')
and  o.addtime <=unix_timestamp('2018-11-12 00:00:00')
改写为：
select 多个字段 from (select id from products where pid='2726' or rootid='2726' union select 2726) s straight_join orderproducts p on
p.cateid=s.id 多表连接 where  o.addtime >= unix_timestamp('2018-10-12 00:00:00')
and  o.addtime <=unix_timestamp('2018-11-12 00:00:00')
执行计划发现外层查询结果较多，select字段中多处标量子查询，把子查询变成left join：
select 多个字段 from (select id from products where pid='2726' or rootid='2726' union select 2726) s straight_join orderproducts p on
p.cateid=s.id 多表连接 left join ..on ..left join .. on ..where  o.addtime >= unix_timestamp('2018-10-12 00:00:00')
and  o.addtime <=unix_timestamp('2018-11-12 00:00:00')
执行计划当中发现where条件数据量较多，回表产生大量IO，创建联合索引，使用ICP特性，减少IO：
create index_t1 on orders(id,addtime)

案例2：
left join中含有union导致的额外排序
select atr.id,tt.tagcode,tt.tripDays,i.membername,i.memberid from
atr
left join avc on atr.topicid=avc.topicid and atr.topicsrc =avc.type
left join i on atr.topicsrc =2 and i.id=atr.topicid
left join (select topic_id topicid,'2' topictype,tag_code tagcode,NULL tripDays,'' startDate from tt1
          union 
          select id,'0',tagdict,trip_days,create_time startDate from tt2) tt
on tt.topictype = atr.topicid
where ...
order by recommendTime Desc limit 1;


改写后：因为外层循环有limit 1限制结果集，可以使用标量子查询修改
select atr.id,
       case when atr.topicsrc =2 then (select tag_code from tt1 where tt1.topic_id=atr.topicid limit 1) 
            when atr.topicsrc =0 then (select tagdict from tt2 when tt2.id=atr.topicid ) end tagcade,
       case when atr.topicsrc =2 then NULL 
            when atr.topicsrc =o then (select trip_days from tt2 where id= atr.topicid) end tripdays,
       i.membername,i.memberid from
       atr
left join avc on atr.topicid=avc.topicid and atr.topicsrc =avc.type
left join i on atr.topicsrc =2 and i.id=atr.topicid
where ...
order by recommendTime Desc limit 1;
```



## nested loop的性能瓶颈

特点：

1 顺序性

2 驱动表的结果集决定连接次数

驱动表必须选择过滤后数据最少的表作为驱动表

```sql
案例1：
优化先join在聚合的SQL，把聚合提前，减少连接的次数，达到优化的目的
mysql> select t.dept_no,count(distinct t.emp_no) from t_group5 t join employees e on t.emp_no=e.emp_no where to_date='9999-01-01' group by t.dept_no;
+---------+--------------------------+
| dept_no | count(distinct t.emp_no) |
+---------+--------------------------+
| d004    |                        1 |
| d005    |                        3 |
| d006    |                        1 |
| d007    |                        1 |
+---------+--------------------------+
4 rows in set (1.31 sec)

mysql> select t.dept_no,count(t.emp_no) from (select t.dept_no,t.emp_no from t_group5 t where to_date='9999-01-01' group by t.dept_no,t.emp_no) t
    -> join  employees e on e.emp_no=t.emp_no
    -> group by t.dept_no;
+---------+-----------------+
| dept_no | count(t.emp_no) |
+---------+-----------------+
| d004    |               1 |
| d005    |               3 |
| d006    |               1 |
| d007    |               1 |
+---------+-----------------+
4 rows in set (0.80 sec)

mysql> desc select t.dept_no,count(distinct t.emp_no) from t_group5 t join employees e on t.emp_no=e.emp_no where to_date='9999-01-01' group by t.dept_no;
+----+-------------+-------+------------+--------+-------------------------+---------+---------+--------------------+---------+----------+--------------------------+
| id | select_type | table | partitions | type   | possible_keys           | key     | key_len | ref                | rows    | filtered | Extra                    |
+----+-------------+-------+------------+--------+-------------------------+---------+---------+--------------------+---------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | ref    | ix_empno_to_date,idx_t2 | idx_t2  | 3       | const              | 1224192 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | e     | NULL       | eq_ref | PRIMARY                 | PRIMARY | 4       | employees.t.emp_no |       1 |   100.00 | Using index              |
+----+-------------+-------+------------+--------+-------------------------+---------+---------+--------------------+---------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> desc select t.dept_no,count(t.emp_no) from (select t.dept_no,t.emp_no from t_group5 t where to_date='9999-01-01' group by t.dept_no,t.emp_no) t join  employees e on e.emp_no=t.emp_no group by t.dept_no;
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+---------------------------------+
| id | select_type | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows    | filtered | Extra                           |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+---------------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 1224192 |   100.00 | Using temporary; Using filesort |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | t.emp_no |       1 |   100.00 | Using index                     |
|  2 | DERIVED     | t          | NULL       | ref    | idx_t2        | idx_t2  | 3       | const    | 1224192 |   100.00 | Using where; Using index        |
+----+-------------+------------+------------+--------+---------------+---------+---------+----------+---------+----------+---------------------------------+
3 rows in set, 1 warning (0.01 sec)
总结：
1 驱动表要小或者有良好的过滤条件使得最终的数据要少
2 驱动表的过滤条件必须要有索引
3 被驱动表的连接条件上要有索引
4 如果驱动表上有重复值，需要使用group by或者distinct去掉

案例2：
减少DBlink的连接次数：

select a.col ....b.corporation_id from 
mego_helper.trade_order a inner join mego_helper.rent_content b
on a.rent_content_id=b.id
where b.sort_id=114 and b.corporation_id in 
(select id from mego_helper.corporation where parent_id=2951) 
and a.update_time > (select max(update_time) from mego.trade_order);
注释：mego.trade_order通过dblink连接
优化思路：先把mego.trade_order数据放在本地 --使用join把mego里数据一次性读到本地
select a.col ....b.corporation_id from 
mego_helper.trade_order a inner join mego_helper.rent_content b
on a.rent_content_id=b.id
inner join (select max(update_time) update_time from mego.trade_order) c
inner join (select id from mego_helper.corporation where parent_id=2951) d 
where b.sort_id=114 and b.corporation_id=d.id
and  a.update_time > c.update_time;
```

3 random access

4 受到连接条件的影响（索引）

被驱动表必须含有连接条件的索引

普通的Nested-Loop Join算法一次只能将一行数据传入内存循环，所以外层循环结果集有多少行，那么内存循环就要执行多少次。



#### Block Nested-Loop Join算法

```sql 
BNL算法原理：将外层循环的行/结果集存入join buffer，内存循环的每一行数据与整个buffer中的记录做比较，可以减少内层循环的扫描次数
举个简单的例子：外层循环结果集有1000行数据，使用NLJ算法需要扫描内层表1000次，但如果使用BNL算法，
则先取出外层表结果集的100行存放到join buffer, 然后用内层表的每一行数据去和这100行结果集做比较，可以一次性与100行数据进行比较，这样内层表其实只需要循环1000/100=10次，减少了9/10。
我们可以从explain的sql执行计划中看到Extra有“using join buffer”，则说明mysql使用了join buffer来对sql进行优化.

使用join buffer的要点：
1）、join buffer size变量大小决定了buffer的大小
2）、只有在join类型为all 或者index或者range的时候，才可以使用join buffer（也就是说explain执行计划中，type为all或者index或者range的时候，才会出现Using join buffer）
--join buffer太小导致的SQL问题
```



## 子查询in，exists，join什么时候用

### in改写为join:

```sql
连接条件M:1直接改写，t emp_no+dept_no PK
                  e emo_no PK  所以是M:1
mysql> select * from t_group t where t.emp_no in (select e.emp_no from employees e) ;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.00 sec)

mysql> select t.* from t_group t join employees e on t.emp_no=e.emp_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.00 sec)

连接条件M:N t emp_no+dept_no PK
           e emp_no+dept_no PK
需要加上distinct
mysql> select * from t_group t where t.emp_no in (select e.emp_no from t_order e) ;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
9 rows in set (0.00 sec)

mysql> select t.* from t_group t join (select distinct e.emp_no from t_order e) e on e.emp_no=t.emp_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
9 rows in set (0.00 sec)
--distinct后结果集较小，把它作为驱动表
mysql> select t.* from (select distinct e.emp_no from t_order e) e straight_join t_group t  on e.emp_no=t.emp_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
9 rows in set (0.00 sec)

案例：
修改SQL里面in子句包含union生成dependent subquery改写成join：
select * from card
where card_no in 
(select card_no from user_profile where user_id is NULL and email ='215782167@qq.com'
union all
select card_no from user_profile where 1=1 and user_id=1998 or mobile = '13611882418');
此语句运行很慢，外层语句rows达到1500W 改写如下：

select c.* from card c join 
(select card_no from user_profile where user_id is NULL and email ='215782167@qq.com'
union ---此处改成union达到去重的目的
select card_no from user_profile where 1=1 and user_id=1998 or mobile = '13611882418') d
on c.card_no=d.card_no


```

### exists 一般用于select当中没有exists子句的列的时候

连接条件是1：M且exists子句数据量较大的时候，要有索引

```sql
mysql> select count(0) from salaries;
+----------+
| count(0) |
+----------+
|  2844047 |
+----------+
1 row in set (7.45 sec)

mysql> select * from t_group t where exists (select 1 from salaries s where s.emp_no=t.emp_no);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.04 sec)

改写成join后很慢
mysql> select t.* from (select distinct s.emp_no from salaries s) s straight_join t_group t on t.emp_no=s.emp_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (1.10 sec)
```



## 多表查询

```SQL

--dept_emp表中求每个部门从1985-01-01开始，每个月的在职人数
mysql> create table t_date(id int primary key auto_increment,yymmdd date);
Query OK, 0 rows affected (0.02 sec)

mysql> insert into t_date(yymmdd) values ('1985-01-01');
Query OK, 1 row affected (0.01 sec)

mysql> select * from t_date;
+----+------------+
| id | yymmdd     |
+----+------------+
|  1 | 1985-01-01 |
+----+------------+
1 row in set (0.00 sec)

mysql> insert into t_date(yymmdd) select adddate('1985-01-01',interval rn day) from (
    -> select @a:=@a+1 rn from (select * from dept_emp limit 10000) a,(select @a:=0) b) c;
Query OK, 10000 rows affected (0.14 sec)
Records: 10000  Duplicates: 0  Warnings: 0

通过中间表产生复制数据
mysql> select dept_no,t.yymmdd,emp_no,d.from_date,d.to_date from dept_emp d,t_time t
    -> where d.from_date <= t.yymmdd
    -> and d.to_date >= t.yymmdd
    -> and t.yymmdd <= '1985-02-28' order by 1,3,2 limit 10;
+---------+------------+--------+------------+------------+
| dept_no | yymmdd     | emp_no | from_date  | to_date    |
+---------+------------+--------+------------+------------+
| d001    | 1985-02-28 |  12025 | 1985-02-28 | 1990-06-09 |
| d001    | 1985-02-13 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-14 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-15 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-16 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-17 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-18 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-19 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-20 |  14227 | 1985-02-13 | 1986-10-02 |
| d001    | 1985-02-21 |  14227 | 1985-02-13 | 1986-10-02 |
+---------+------------+--------+------------+------------+
10 rows in set (1.75 sec)

mysql> select dept_no,date_format(t.yymmdd,'%Y%m'),count(distinct emp_no) from
    -> dept_emp d,t_time t
    -> where d.from_date<= t.yymmdd
    -> and d.to_date >= t.yymmdd
    ->  and t.yymmdd <= '1985-02-28'
    -> group by dept_no,date_format(t.yymmdd,'%Y%m');
+---------+------------------------------+------------------------+
| dept_no | date_format(t.yymmdd,'%Y%m') | count(distinct emp_no) |
+---------+------------------------------+------------------------+
| d001    | 198501                       |                      1 |
| d001    | 198502                       |                     86 |
| d002    | 198501                       |                      2 |
| d002    | 198502                       |                     83 |
| d003    | 198501                       |                      1 |
| d003    | 198502                       |                     76 |
| d004    | 198501                       |                      1 |
| d004    | 198502                       |                    335 |
| d005    | 198501                       |                      1 |
| d005    | 198502                       |                    450 |
| d006    | 198501                       |                      1 |
| d006    | 198502                       |                     70 |
| d007    | 198501                       |                      1 |
| d007    | 198502                       |                    235 |
| d008    | 198501                       |                      1 |
| d008    | 198502                       |                     93 |
| d009    | 198501                       |                      1 |
| d009    | 198502                       |                     92 |
+---------+------------------------------+------------------------+
18 rows in set (1.82 sec)

--延迟join的案例：
先后排序改写 结果集不变
mysql> select t.* from t_group t straight_join (select distinct e.emp_no from t_order e) e on t.emp_no=e.emp_no order by t.emp_no desc ;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
9 rows in set (0.08 sec)

mysql> select t.* from (select t.* from t_group t order by t.emp_no desc limit 100) t
    -> straight_join (select distinct e.emp_no from t_order e) e
    -> on t.emp_no=e.emp_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
9 rows in set (0.00 sec)

select a.字段 ... c.username,c.sub_config_id from
tp_store a inner join tp_wx b on b.token=a.token
inner join tp_user as c on b.uid=c.id
where a.exam_status =1
and (c.sub_config_id=0) 
order by a.store_id desc limit 15;
思路：根据where条件查询 发现a.exam_status =1的选择率较好，以a表作为驱动表，使用延迟join，减少join的数据量

改写如下：
select a.字段 ... c.username,c.sub_config_id from 
(select a.store_id,c.id from (select distinct a.store_id,a.token from tp_store a where a.exam_status = 1 order by a.store_id desc) as a
straight_join tp_wx b on a.token=b.token 
straight_join tp_user c on b.uid=c.id where 1=1 and (c.sub_config_id=0) limit 15) b straight_join tp_store a 
on a.store_id=b.store_id
straight_join tp_user c on b.uid=c.id

在access条件上exam_status 列上创建索引

总结：SQL末端含有limit 分页，可以使用延迟join




```

