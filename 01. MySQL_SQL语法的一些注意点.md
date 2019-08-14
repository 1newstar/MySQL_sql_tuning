[TOC]

# MySQL    SQL语法的一些注意点

## 常用函数

### IFNULL   ISNULL  和  NULL的处理

注意：mysql当中 ' '  不等于NULL，这与Oracle当中不同！

例：

```mssql
mysql> select * from test1;
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
| NULL | NULL |
|      | sad  |
+------+------+
mysql> select * from test1 where id is not null;
+------+------+
| id   | n    |
+------+------+
|      | sad  |
| 1000 | NULL |
+------+------+
mysql> select * from test1 where id is not null and id != '';
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
+------+------+


--IFNULL作为空值的筛选
mysql> select IFNULL(null,1);
+----------------+
| IFNULL(null,1) |
+----------------+
|              1 |
+----------------+
1 row in set (0.01 sec)

mysql> select IFNULL('',1);
+--------------+
| IFNULL('',1) |
+--------------+
|              |
+--------------+
1 row in set (0.00 sec)



--ISNULL判断是否是空值，是空值取值1：
mysql> select ISNULL(NULL),ISNULL(0),ISNULL(1/0);
+--------------+-----------+-------------+
| ISNULL(NULL) | ISNULL(0) | ISNULL(1/0) |
+--------------+-----------+-------------+
|            1 |         0 |           1 |
+--------------+-----------+-------------+
--注意：在Oracle当中1/0是报错的，mysql是取NULL！

mysql> select 1=1,null=null,1=null,null is null,1<>null;
+-----+-----------+--------+--------------+---------+
| 1=1 | null=null | 1=null | null is null | 1<>null |
+-----+-----------+--------+--------------+---------+
|   1 |      NULL |   NULL |            1 |    NULL |
+-----+-----------+--------+--------------+---------+

--对于NULL的两值的判断处理
mysql> select 1<=>1,null<=>null;
+-------+-------------+
| 1<=>1 | null<=>null |
+-------+-------------+
|     1 |           1 |
+-------+-------------+
1. 和=号的相同点，像常规的=运算符一样，两个值进行比较，结果是0（不等于）或1（相等）;换句话说：'A'<=>'B'得0 和 'a'<=> 'a'得1。
2. 和=号的不同点
   和=运算符不同的是，NULL的值是没有任何意义的。所以=号运算符不能把NULL作为有效的结果。所以：请使用<=>,
   'a' <=> NULL 得0   NULL<=> NULL 得出 1。和=运算符正相反，=号运算符规则是 'a'=NULL 结果是NULL 甚至NULL = NULL 结果也是NULL。
   顺便说一句，mysql上几乎所有的操作符和函数都是这样工作的，因为和NULL比较基本上都没有意义。
```

### 时间函数    NOW  和  SYSDATE

NOW()取的是语句开始执行的时间，SYSDATE()取的是动态的实时时间。

因为NOW()取自mysql的一个变量”TIMESTAMP”，而这个变量在语句开始执行的时候就设定好了，因此在整个语句执行过程中都不会变化。

```sql
mysql> select now(),sleep(2),now();
+---------------------+----------+---------------------+
| now()               | sleep(2) | now()               |
+---------------------+----------+---------------------+
| 2019-03-27 19:34:12 |        0 | 2019-03-27 19:34:12 |
+---------------------+----------+---------------------+

mysql> select sysdate(),sleep(2),sysdate();
+---------------------+----------+---------------------+
| sysdate()           | sleep(2) | sysdate()           |
+---------------------+----------+---------------------+
| 2019-03-27 19:34:57 |        0 | 2019-03-27 19:34:59 |
+---------------------+----------+---------------------+

--案例分析动态函数使得索引失效：
mysql> desc select * from salaries where emp_no=10001 and from_date > now();
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,emp_no | PRIMARY | 7       | NULL |    1 |   100.00 | Using where |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> desc select * from salaries where emp_no=10001 and from_date > sysdate();
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys  | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | const |   17 |    33.33 | Using where |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
可以看到sysdate函数是动态的，无法使用联合索引！
改写方法：
mysql> desc select * from salaries s,(select sysdate() d) g where emp_no=10001 and from_date > g.d;
+----+-------------+------------+------------+--------+----------------+---------+---------+------+------+----------+----------------+
| id | select_type | table      | partitions | type   | possible_keys  | key     | key_len | ref  | rows | filtered | Extra          |
+----+-------------+------------+------------+--------+----------------+---------+---------+------+------+----------+----------------+
|  1 | PRIMARY     | <derived2> | NULL       | system | NULL           | NULL    | NULL    | NULL |    1 |   100.00 | NULL           |
|  1 | PRIMARY     | s          | NULL       | range  | PRIMARY,emp_no | PRIMARY | 7       | NULL |    1 |   100.00 | Using where    |
|  2 | DERIVED     | NULL       | NULL       | NULL   | NULL           | NULL    | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+------------+------------+--------+----------------+---------+---------+------+------+----------+----------------+
同理，变量函数导致索引无法使用的情况，优化如下：
select guid from bussiness force index(PRI) where guid=func_guid();
改写后：
select guid from (select func_guid() a1 from dual limit 1) a join bussiness b on b.guid=a.al;
**************************************************************



--对索引列进行了运算导致索引失效，时间函数的实际优化案例：
select flag,id,count(flag) as count from doll where date(create_date)=curdate() groupby id,flag order by id desc;
由于对create_date列使用了date函数导致索引无法使用，改写如下：
select flag,id,count(flag) as count from doll where create_date>= concat(curdate(),' 00:00:01') and create_date <= concat(curdate(),' 23:59:59') groupby id,flag order by id desc;

```



### 字符格式化函数

#### LPAD   RPAD    TRIM     RTRIM     LTRIM

```sql
实验：
mysql> select * from t_group;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+

--LPAD使得数字列隐式转为字符串
mysql> select substr(concat(LPAD(emp_no,5,'0'),dept_no),1,5) aa from t_group;
+-------+
| aa    |
+-------+
| 22744 |
| 24007 |
| 03097 |
| 31112 |
| 40983 |
| 46554 |
| 48317 |
| 49667 |
| 50449 |
| 10004 |
+-------+
--加上0使得字符串转为数字
mysql> select substr(concat(LPAD(emp_no,5,'0'),dept_no),1,5)+0 aa from t_group;
+-------+
| aa    |
+-------+
| 22744 |
| 24007 |
|  3097 |
| 31112 |
| 40983 |
| 46554 |
| 48317 |
| 49667 |
| 50449 |
| 10004 |
+-------+
字符类型+0转换为数字类型


mysql> select hex(substr(concat(RPAD(emp_no,5,'0'),dept_no),1,5)) aa from t_group;
+------------+
| aa         |
+------------+
| 3232373434 |
| 3234303037 |
| 3330393730 |
| 3331313132 |
| 3430393833 |
| 3436353534 |
| 3438333137 |
| 3439363637 |
| 3530343439 |
| 3130303034 |
+------------+
10 rows in set (0.00 sec)

mysql> select hex(substr(concat(RPAD(emp_no,5,'0'),dept_no),1,5)+0) aa from t_group;
+------+
| aa   |
+------+
| 58D8 |
| 5DC7 |
| 78FA |
| 7988 |
| A017 |
| B5DA |
| BCBD |
| C203 |
| C511 |
| 2714 |
+------+
通过hex函数查看字符，数字的16进制的表现：
mysql> select hex(100);
+----------+
| hex(100) |
+----------+
| 64       |
+----------+

mysql> select hex('100');
+------------+
| hex('100') |
+------------+
| 313030     |
+------------+
```



#### 字符连接concat

```sql
mysql> select concat('1','b');
+-----------------+
| concat('1','b') |
+-----------------+
| 1b              |
+-----------------+
1 row in set (0.00 sec)

mysql> select '1'||'b';
+----------+
| '1'||'b' |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)


```



### mysql  中数字类型的特殊性

```sql

mysql> select 1+'33a';
+---------+
| 1+'33a' |
+---------+
|      34 |
+---------+
1 row in set, 1 warning (0.00 sec)

mysql> select 'b1'+'33a';
+------------+
| 'b1'+'33a' |
+------------+
|         33 |
+------------+
1 row in set, 2 warnings (0.00 sec)


mysql> select * from test1;
+------+------+
| id   | n    |
+------+------+
| 1000 | NULL |
| NULL | NULL |
|      | sad  |
| 1a   | sa   |
+------+------+
4 rows in set (0.00 sec)

mysql> select *  from test1 where id =1;
+------+------+
| id   | n    |
+------+------+
| 1a   | sa   |
+------+------+
1 row in set, 1 warning (0.04 sec)

mysql> select *  from test1 where id ='1';
Empty set (0.00 sec)

mysql> desc select *  from test1 where id =1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | test1 | NULL       | ALL  | ix_id         | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)

****************************************************************************
案例分析：

mysql> insert into test1 values ('22744asd','saada');
Query OK, 1 row affected (0.00 sec)

mysql> select * from test1;
+----------+-------+
| id       | n     |
+----------+-------+
| 1000     | NULL  |
| NULL     | NULL  |
|          | sad   |
| 1a       | sa    |
| 22744asd | saada |
+----------+-------+
5 rows in set (0.00 sec)

mysql> select * from test1 t,t_order o where t.id=o.emp_no;
+----------+-------+--------+---------+------------+------------+
| id       | n     | emp_no | dept_no | from_date  | to_date    |
+----------+-------+--------+---------+------------+------------+
| 22744asd | saada |  22744 | d006    | 1986-12-01 | 9999-01-01 |
+----------+-------+--------+---------+------------+------------+
1 row in set, 1 warning (0.07 sec)
--字符类型隐式转换成数字类型
利用concat转换数字类型为字符类型：
--数字与''连接转换成字符类型
mysql> select * from test1 t,t_order o where t.id=concat(o.emp_no,'');
Empty set (0.00 sec)

mysql> desc select * from test1 t,t_order o where t.id=concat(o.emp_no,'');
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | ix_id         | NULL | NULL    | NULL |    5 |   100.00 | NULL                                               |
|  1 | SIMPLE      | o     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> desc select * from test1 t,t_order o where t.id=o.emp_no;
+----+-------------+-------+------------+------+---------------+-------+---------+----------------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key   | key_len | ref            | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------+---------+----------------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | ix_id         | NULL  | NULL    | NULL           |    5 |   100.00 | Using where           |
|  1 | SIMPLE      | o     | NULL       | ref  | ix_t1         | ix_t1 | 5       | employees.t.id |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------+---------+----------------+------+----------+-----------------------+
2 rows in set, 2 warnings (0.00 sec)

发生隐式转换导致SQL变慢，使用show warnings查看执行计划：
mysql> show warnings\G;
*************************** 1. row ***************************
  Level: Warning
   Code: 1739
Message: Cannot use ref access on index 'ix_id' due to type or collation conversion on field 'id'
*************************** 2. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `employees`.`t`.`id` AS `id`,`employees`.`t`.`n` AS `n`,`employees`.`o`.`emp_no` AS `emp_no`,`employees`.`o`.`dept_no` AS `dept_no`,`employees`.`o`.`from_date` AS `from_date`,`employees`.`o`.`to_date` AS `to_date` from `employees`.`test1` `t` join `employees`.`t_order` `o` where (`employees`.`t`.`id` = `employees`.`o`.`emp_no`)
2 rows in set (0.00 sec)

隐式转换的例子：
例子Eg1：
执行计划extra部分包含：range checked for each record常常说明发生了隐式转换，导致无法使用相关索引。
改写方法：将非索引列取值通过concat等手段转换成索引列的数据类型！！

例子Eg2：
MySQL表字符集不一样导致性能问题
改写方法：将连接条件当中，非索引使用列改写成和目的使用索引列相同的字符类型。
如：
on convert(t.order_num using utf8) = o.order_id
```

​        例子Eg1：

执行计划extra部分包含：range checked for each record常常说明发生了隐式转换，导致无法使用相关索引。

改写方法：将非索引列取值通过concat等手段转换成索引列的数据类型！！



​        例子Eg2：

MySQL表字符集不一样导致性能问题

改写方法：将连接条件当中，非索引使用列改写成和目的使用索引列相同的字符类型。

如：

on convert(t.order_num using utf8) = o.order_id



### 日期类型的特殊性

别的数据库，日期类型必须转换，但是mysql不需要，他会自动转换。

只要书写对应的格式是日期类型即可：

```sql
mysql> desc select emp_no from salaries where emp_no=20247 and from_date='1985-03-01';
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref         | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | const | PRIMARY,emp_no | PRIMARY | 7       | const,const |    1 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> desc select emp_no from salaries where emp_no=20247 and from_date=str_to_date('1985-03-01','%Y-%m-%d');
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref         | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | const | PRIMARY,emp_no | PRIMARY | 7       | const,const |    1 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> desc select emp_no from salaries where emp_no=20247 and from_date='19850301';
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref         | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | const | PRIMARY,emp_no | PRIMARY | 7       | const,const |    1 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> desc select emp_no from salaries where emp_no=20247 and from_date=19850301;
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref         | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | const | PRIMARY,emp_no | PRIMARY | 7       | const,const |    1 |   100.00 | Using index |
+----+-------------+----------+------------+-------+----------------+---------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```



### between  和 in 的区别  和 相同

对于字符类型，尽量避免使用between   and；即使要使用，不要作为联合索引前导列的筛选条件！



### like的用法和特点

```sql
对于字符类型的列，使用 like 'AA%' 有可能使用一些索引
对于日期类型的列，使用 like '1986%' 无法使用索引
对于数字类型的列，使用 like '10009%' 无法使用索引
```



### 选择率

重复值越少 ，选择率越好

### OR 的使用注意点

```SQL
使用or必须养成两边添加括号的习惯，否则结果完全不一样！

案例1：
标量子查询中筛选条件中or导致autokey无法生成改写：
pc.count=0 or pc.count is null
==> 改写为：case when pc.count is null then 0 else pc.count end =0

案例2：
使用union打开or
```



### in  和  not in

```SQL
in 数据类型要一致，相当于or,无法识别null

select t1.* from test1 t1,(select distinct id from test1) t2 where t1.id <=> t2.id;   
利用<=>来取得NULL值

not in 无法识别空值
案例：
not in 改写成 left join
select * from t_order where emp_no not in (select emp_no from employees where emp_no > 20000);
--emp_no里面有NULL值
改写为：

mysql> select t.* from t_order t left join (select emp_no from employees where emp_no > 20000) s  on t.emp_no=s.emp_no where s.emp_no is null and t.emp_no is not null;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
```



​      

### exists , not exists

```sql
案例1：

exists改写 in：针对外层结果，驱动表结果集比较小，效果不错；
              但还是要注意循环内部，where条件的索引问题。
not  exists 改写 （）=0子查询： where子句中（count（*））子查询结果为0表示不存在的条件，可以使用not exists改写：
where (select count(id) from s where s.date < '2017-09-01' and s.id = a.id) =0   ==>  
where not exists (select 1 from s where  s.date < '2017-09-01' and s.id=a.id)

     
案例2：
exists 改写成 left join： 主要针对外层结果集数量较大的情况。  改写思路：
缩小外层结果集的数量，使用left join来改写！


```



### 使用group by提前来减少外层结果集

```sql
优化前：
外循环结果集过大，SQL结果运行无果，想办法减少外层结果集        

SELECT p.channel_id 渠道,count(p.pur_number) 订单量, SUM(p.pay) 金额 
FROM sales_order s,pur_order p 
WHERE s.sales_id=p.sales_id 
and p.split_type != 'SPLIT_SUB'
and p.related_order_number is null
and s.order_time >= '2018-08-01' 
and s.order_time <= '2019-01-01'
and s.store_id='S001'
and (s.type=0 or (s.type!=0 AND s.payment_status =1))
AND NOT EXISTS
(select 1 from roc where roc.order_id = p.pur_number AND roc.month='2018-08' and order_id<>'')
GROUP BY p.channel_id ;



优化后：
SELECT p.channel_id 渠道,sum(p.c1) 订单量, SUM(p.s1) 金额
FROM (select p.channel_id,p.pur_number,count(p.pur_number) c1,SUM(p.pay) s1 
from sales_order s,pur_order p where s.sales_id=p.sales_id 
and p.split_type != 'SPLIT_SUB'
and p.related_order_number is null
and s.order_time >= '2018-08-01' 
and s.order_time <= '2019-01-01'
and s.store_id='S001'
and (s.type=0 or (s.type!=0 AND s.payment_status =1))
group by p.channel_id,p.pur_number) p 
LEFT JOIN 
(select distinct roc.order_id from  roc where 1=1 and roc.month='2018-08' and order_id<>'' limit 1000000 ) roc 
ON roc.order_id=p.pur_number
WHERE roc.order_id is NULL
GROUP BY p.channel_id ;

此处有两个技巧： 
1.group by提前减少结果集
2.使用left join连接，使用distinct去重，是的左边表和右边表的结果集是多对一，连接之后，右边表缺少的值用NULL来补充，
业务需求就是筛选出外层结果集在内层中不存在的结果，故where条件选择连接后roc.order_id是NULL的结果：
mysql>  select * from t_order t left join (select emp_no from employees where emp_no > 20000) s  on t.emp_no=s.emp_no where s.emp_no is null and t.emp_no is not null;
+--------+---------+------------+------------+--------+
| emp_no | dept_no | from_date  | to_date    | emp_no |
+--------+---------+------------+------------+--------+
|  10004 | d004    | 1986-12-01 | 9999-01-01 |   NULL |
+--------+---------+------------+------------+--------+
```



### 对多列分组后，注意原先的count聚合函数，转换成外层的sum聚合函数

```SQL
--对多列分组后，注意原先的count聚合函数，转换成外层的sum聚合函数
mysql> select * from t_group;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|   3097 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1987-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
11 rows in set (0.00 sec)

mysql> select dept_no,count(1) from t_group group by dept_no;
+---------+----------+
| dept_no | count(1) |
+---------+----------+
| d002    |        1 |
| d004    |        1 |
| d005    |        5 |
| d006    |        1 |
| d007    |        1 |
| d008    |        2 |
+---------+----------+
6 rows in set (0.00 sec)

mysql> select s.dept_no,sum(c1) from (
    -> select dept_no,emp_no,count(1) c1 from t_group group by dept_no,emp_no) s
    -> group by s.dept_no;
+---------+---------+
| dept_no | sum(c1) |
+---------+---------+
| d002    |       1 |
| d004    |       1 |
| d005    |       5 |
| d006    |       1 |
| d007    |       1 |
| d008    |       2 |
+---------+---------+
6 rows in set (0.00 sec)

mysql> select s.dept_no,count(c1) from ( select dept_no,emp_no,count(1) c1 from t_group group by dept_no,emp_no) s group by s.dept_no;
+---------+-----------+
| dept_no | count(c1) |
+---------+-----------+
| d002    |         1 |
| d004    |         1 |
| d005    |         4 |
| d006    |         1 |
| d007    |         1 |
| d008    |         2 |
+---------+-----------+
6 rows in set (0.00 sec)
注意对多列分组后，注意原先的count聚合函数，转换成外层的sum聚合函数
mysql> select dept_no,emp_no,count(1) c1 from t_group group by dept_no,emp_no;
+---------+--------+----+
| dept_no | emp_no | c1 |
+---------+--------+----+
| d002    |  31112 |  1 |
| d004    |  10004 |  1 |
| d005    |   3097 |  1 |
| d005    |  24007 |  1 |
| d005    |  40983 |  1 |
| d005    |  50449 |  2 |
| d006    |  22744 |  1 |
| d007    |  49667 |  1 |
| d008    |  46554 |  1 |
| d008    |  48317 |  1 |
+---------+--------+----+
10 rows in set (0.00 sec)

```



### LIMIT

union  / union all中不能使用limit：可以使用子查询绕过

```sql 
select * from (select emp_no from employees e limit 10) e
union all
select * from (select emp_no from employees e limit 10) b
```







