[TOC]



# MySQL优化：SQL语句的基础

## SQL语句的基本子句

### group by

在mysql 5.6以及之前的版本 会出现不完全分组的group by 语句，而在5.7中会报错

```SQL
mysql> select * from t_group where dept_no='d005' group by dept_no;
ERROR 1055 (42000): Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'employees.t_group.emp_no' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
**5.7版本出现报错**
**修改SQL_mode**
mysql> set sql_mode='';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select * from t_group where dept_no='d005' group by dept_no;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
```

### order by

注意点：

明确写明需要排序的列，不要使用数字代替。有时代码变更，会导致排序的列出错！



### limit 

返回数据，用于分页

```sql
mysql> select * from t_group order by dept_no ;
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
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
+--------+---------+------------+------------+

mysql> select * from t_group order by dept_no limit 2,4;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
可见 limit 2,4（从第三行开始，由此知道计数第0行）
```

### having 

对group by的结果进程过滤

## 子查询

### 1)  from 后面的子查询 必须有别名

```SQL
mysql> select * from (select * from t_group order by dept_no limit 2,4);
ERROR 1248 (42000): Every derived table must have its own alias
mysql> select * from (select * from t_group order by dept_no limit 2,4) x;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
```

### 2）标量子查询  是指select 和 from 之间使用的子查询

注意：标量子查询的结果只能返回一行

```sql
mysql> select t.*,(select e.emp_no from employees e where e.emp_no=t.emp_no) s from t_group t;
+--------+---------+------------+------------+-------+
| emp_no | dept_no | from_date  | to_date    | s     |
+--------+---------+------------+------------+-------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 | 22744 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 | 24007 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 | 30970 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 | 31112 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 | 40983 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 | 46554 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 | 48317 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 | 49667 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 | 50449 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 | 10004 |
+--------+---------+------------+------------+-------+
```

### in 和 exists

```sql
mysql> select * from t_group t where t.emp_no in (select e.emp_no from employees e);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.09 sec)

mysql> select * from t_group t where exists (select e.emp_no from employees e where e.emp_no=t.emp_no);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.05 sec)
```

### not exists

注意：如果列里面有NULL会导致no in的结果不正确

```SQL
mysql> select * from t_group where emp_no not in (null);
Empty set (0.00 sec)

mysql> select * from t_group where not exists (select emp_no from t_group where emp_no is null);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+




mysql> select * from t_group where emp_no not in (select emp_no from t_order );
Empty set (0.00 sec)
因为有null导致not in的查询结果有问题
--去掉null结果正常
mysql> select * from t_group where emp_no not in (select o.emp_no from t_order o where o.emp_no is not null);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
1 row in set (0.00 sec)

mysql> select * from t_group t where not exists (select emp_no from t_order o where o.emp_no = t.emp_no);
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
+--------+---------+------------+------------+
```



## 表连接

### 笛卡尔积：表的行数相乘

产生笛卡尔积标志：在执行计划当中看到ALL 和Using join buffer的组合，要当心笛卡尔积！

```SQL
mysql> desc select * from dept_emp,departments\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: departments
   partitions: NULL
         type: index
possible_keys: NULL
          key: dept_name
      key_len: 122
          ref: NULL
         rows: 9
     filtered: 100.00
        Extra: Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 331143
     filtered: 100.00
        Extra: Using join buffer (Block Nested Loop)
2 rows in set, 1 warning (0.00 sec)

ERROR: 
No query specified

mysql> desc select * from dept_emp t,departments d where t.dept_no=d.dept_no\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: d
   partitions: NULL
         type: index
possible_keys: PRIMARY
          key: dept_name
      key_len: 122
          ref: NULL
         rows: 9
     filtered: 100.00
        Extra: Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: t
   partitions: NULL
         type: ref
possible_keys: dept_no
          key: dept_no
      key_len: 12
          ref: employees.d.dept_no
         rows: 41392
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.00 sec)

产生笛卡尔积标志：在执行计划当中看到ALL 和Using join buffer的组合，要当心笛卡尔积！
```

### 等值join 和between join

```sql
来查询dept_no为d008的组的人员上班天数
mysql> select count(*) from t_time;
+----------+
| count(*) |
+----------+
|    20000 |
+----------+
1 row in set (0.01 sec)

mysql> select * from t_time order by 2 desc limit 10;
+-------+------------+
| id    | yymmdd     |
+-------+------------+
| 20000 | 2039-10-05 |
| 19999 | 2039-10-04 |
| 19998 | 2039-10-03 |
| 19997 | 2039-10-02 |
| 19996 | 2039-10-01 |
| 19995 | 2039-09-30 |
| 19994 | 2039-09-29 |
| 19993 | 2039-09-28 |
| 19992 | 2039-09-27 |
| 19991 | 2039-09-26 |
+-------+------------+
10 rows in set (0.07 sec) 
注释：在开发中，这张表称为字典表。

mysql> select * from t_group;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  22744 | d006    | 1986-12-01 | 9999-01-01 |
|  24007 | d005    | 1986-12-01 | 9999-01-01 |
|  30970 | d005    | 1986-12-01 | 2017-03-29 |
|  31112 | d002    | 1986-12-01 | 1993-12-10 |
|  40983 | d005    | 1986-12-01 | 9999-01-01 |
|  46554 | d008    | 1986-12-01 | 1992-05-27 |
|  48317 | d008    | 1986-12-01 | 1989-01-11 |
|  49667 | d007    | 1986-12-01 | 9999-01-01 |
|  50449 | d005    | 1986-12-01 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (0.00 sec)

mysql> select  g.emp_no,count(1) from t_group g,t_time t where g.dept_no='d008'
    -> and t.yymmdd >= g.from_date
    -> and t.yymmdd <= g.to_date
    -> group by g.emp_no
    -> order by g.emp_no;
+--------+----------+
| emp_no | count(1) |
+--------+----------+
|  46554 |     2005 |
|  48317 |      773 |
+--------+----------+
2 rows in set (0.07 sec)
```

### join 的简单案例

on 连接的两列中的值是N：N的关系，会在结果中产生NULL值；连接的两列中有重复值就会产生重复值，见如下的例子：

```sql 
表1：
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
表2：
mysql> select * from dept2;
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
18 rows in set (0.00 sec)

表3：
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
| d010    | Sageyan            |
| d007    | Sales              |
+---------+--------------------+
10 rows in set (0.00 sec)

左连接left join:
mysql> select * from departments d inner join dept2 t on d.dept_no=t.dept_no;
+---------+--------------------+---------+--------------------+
| dept_no | dept_name          | dept_no | dept_name          |
+---------+--------------------+---------+--------------------+
| d009    | Customer Service   | d009    | Customer Service   |
| d005    | Development        | d005    | Development        |
| d002    | Finance            | d002    | Finance            |
| d003    | Human Resources    | d003    | Human Resources    |
| d001    | Marketing          | d001    | Marketing          |
| d004    | Production         | d004    | Production         |
| d006    | Quality Management | d006    | Quality Management |
| d008    | Research           | d008    | Research           |
| d007    | Sales              | d007    | Sales              |
| d009    | Customer Service   | d009    | Customer Service   |
| d005    | Development        | d005    | Development        |
| d002    | Finance            | d002    | Finance            |
| d003    | Human Resources    | d003    | Human Resources    |
| d001    | Marketing          | d001    | Marketing          |
| d004    | Production         | d004    | Production         |
| d006    | Quality Management | d006    | Quality Management |
| d008    | Research           | d008    | Research           |
| d007    | Sales              | d007    | Sales              |
+---------+--------------------+---------+--------------------+
18 rows in set (0.00 sec)

mysql>  select * from departments d left join dept2 t on d.dept_no=t.dept_no;
+---------+--------------------+---------+--------------------+
| dept_no | dept_name          | dept_no | dept_name          |
+---------+--------------------+---------+--------------------+
| d009    | Customer Service   | d009    | Customer Service   |
| d005    | Development        | d005    | Development        |
| d002    | Finance            | d002    | Finance            |
| d003    | Human Resources    | d003    | Human Resources    |
| d001    | Marketing          | d001    | Marketing          |
| d004    | Production         | d004    | Production         |
| d006    | Quality Management | d006    | Quality Management |
| d008    | Research           | d008    | Research           |
| d007    | Sales              | d007    | Sales              |
| d009    | Customer Service   | d009    | Customer Service   |
| d005    | Development        | d005    | Development        |
| d002    | Finance            | d002    | Finance            |
| d003    | Human Resources    | d003    | Human Resources    |
| d001    | Marketing          | d001    | Marketing          |
| d004    | Production         | d004    | Production         |
| d006    | Quality Management | d006    | Quality Management |
| d008    | Research           | d008    | Research           |
| d007    | Sales              | d007    | Sales              |
| d010    | Sageyan            | NULL    | NULL               |
+---------+--------------------+---------+--------------------+
19 rows in set (0.00 sec)

inner join
mysql> select * from departments d inner join dept2 t on d.dept_no=t.dept_no;
+---------+--------------------+---------+--------------------+
| dept_no | dept_name          | dept_no | dept_name          |
+---------+--------------------+---------+--------------------+
| d009    | Customer Service   | d009    | Customer Service   |
| d005    | Development        | d005    | Development        |
| d002    | Finance            | d002    | Finance            |
| d003    | Human Resources    | d003    | Human Resources    |
| d001    | Marketing          | d001    | Marketing          |
| d004    | Production         | d004    | Production         |
| d006    | Quality Management | d006    | Quality Management |
| d008    | Research           | d008    | Research           |
| d007    | Sales              | d007    | Sales              |
| d009    | Customer Service   | d009    | Customer Service   |
| d005    | Development        | d005    | Development        |
| d002    | Finance            | d002    | Finance            |
| d003    | Human Resources    | d003    | Human Resources    |
| d001    | Marketing          | d001    | Marketing          |
| d004    | Production         | d004    | Production         |
| d006    | Quality Management | d006    | Quality Management |
| d008    | Research           | d008    | Research           |
| d007    | Sales              | d007    | Sales              |
+---------+--------------------+---------+--------------------+
18 rows in set (0.00 sec)
```

## 创建PK的案例

```sql
mysql> select t.emp_no,t.dept_no,count(1) from t_group4 t group by t.emp_no,t.dept_no having count(1) > 1;
+--------+---------+----------+
| emp_no | dept_no | count(1) |
+--------+---------+----------+
|  10004 | d004    |   262144 |
|  22744 | d006    |   262144 |
|  24007 | d005    |   262144 |
|  30970 | d005    |   262144 |
|  31112 | d002    |   262144 |
|  40983 | d005    |   262144 |
|  46554 | d008    |   262144 |
|  48317 | d008    |   262144 |
|  49667 | d007    |   262144 |
|  50449 | d005    |   262144 |
+--------+---------+----------+
不是唯一的，不能创建PK 这条SQL用于检查能否创建主键;这两列不能创建PK
```

## 创建FK的案例

```sql
mysql> select count(1) from t_group4 t where not exists (select emp_no from employees e where  e.emp_no=t.emp_no);
+----------+
| count(1) |
+----------+
|        0 |
+----------+
用来检查前表能否创建指向后表的外键，没有选择行，说明前表的列是后表的子集！
```

