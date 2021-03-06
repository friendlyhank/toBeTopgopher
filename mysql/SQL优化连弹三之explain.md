﻿MySQL提供了EXPLAIN或DESC命令,获取MYSQL如何执行SELECT语句得信息,包括在SELECT语句执行过程中表如何连接和连接得顺序。

为了方便演示，大家可以下载官方[sakila示例数据库文件](https://dev.mysql.com/doc/index-other.html)

```sql
mysql> explain select * from film;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | film  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```
 - **select_type**表示select的类型,常见的取值有SIMPLE(简单表，即不使用表连接或者子查询)、PRIMARY(主查询，即外层的查询)、UNION、SUBQUERY(子查询的第一个SELECT)
 - **table**表示对应的表
 - **partitions** 表的分区
 - **possible_keys** 此次查询可能用到的索引
 - **key** 此次查询确切用的索引
 - **key_len** 使用到索引字段得长度
 - **rows**扫描行的数量
 - **filtered**此次查询条件所过滤数据的百分比
 - **extra**执行情况的说明和描述
 - **type**表示MySQL在表中找到所需行的方式，或者叫访问类型。
 
接下来我们具体根据type进行讲解,其次会根据多角度切入分析具体实际的例子。
## TYPE
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200520180848914.png)
从左到右，性能由差变好。

## NULL
type=null,MySQL不用访问表或者索引，直接就能够得到结果。
```sql
mysql> EXPLAIN select 1 from dual;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```

## const/system
type=const/system,const是根据**主键索引primary key或者唯一索引unique index**进行查询,system表示系统只有一条数据，这是一个特殊得const类型。**单表中最多一个匹配行**，查询起来非常迅速。

(1) film_id为主键索引
```sql
mysql> EXPLAIN select * from film where film_id=1;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | film  | NULL       | const | PRIMARY       | PRIMARY | 2       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

(2)连表查询 ,film.film_id和film_text.film_id都为主键索引
```sql
mysql> EXPLAIN select * from film INNER JOIN film_text  on film.film_id = film_text.film_id where film.film_id=1;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | film      | NULL       | const | PRIMARY       | PRIMARY | 2       | const |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | film_text | NULL       | const | PRIMARY       | PRIMARY | 2       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
2 rows in set, 1 warning (0.01 sec)
```
## eq_ref
type=eq_ref,类似于ref,区别在使用索引是唯一索引，简单来说，是根据**主键索引primary key或者唯一索引unique index**进行查询。

连表查询 ,film.film_id和film_text.film_id都为主键索引
```sql
mysql> EXPLAIN select * from film INNER JOIN film_text on film.film_id=film_text.film_id;
+----+-------------+-----------+------------+--------+---------------+---------+---------+---------------------+------+----------+-------------+
| id | select_type | table     | partitions | type   | possible_keys | key     | key_len | ref                 | rows | filtered | Extra       |
+----+-------------+-----------+------------+--------+---------------+---------+---------+---------------------+------+----------+-------------+
|  1 | SIMPLE      | film      | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL                | 1000 |   100.00 | NULL        |
|  1 | SIMPLE      | film_text | NULL       | eq_ref | PRIMARY       | PRIMARY | 2       | sakila.film.film_id |    1 |   100.00 | Using where |
+----+-------------+-----------+------------+--------+---------------+---------+---------+---------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
mysql在底层会做优化，会自动找出数据项较少的表，先进行全表的扫描，然后再根据主键索引与另外一张表建立连接。

## ref非唯一索引扫描
type=ref,使用**非唯一索引扫描**,返回匹配某个单独值得记录行。

(1)customer_id为非唯一索引
```sql
mysql> EXPLAIN select * from payment where customer_id=1;
+----+-------------+---------+------------+------+--------------------+--------------------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys      | key                | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+--------------------+--------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | payment | NULL       | ref  | idx_fk_customer_id | idx_fk_customer_id | 2       | const |   32 |   100.00 | NULL  |
+----+-------------+---------+------------+------+--------------------+--------------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
注1：如果匹配值得记录行为所有记录，type会记为All
注2:主键索引为特殊的唯一索引，所以这里用主键去匹配值得时候执行计划返回的type不是ref。

(2)type=ref也常常出现在join操作中：
```sql
mysql> EXPLAIN select * from customer INNER JOIN payment on customer.customer_id=payment.customer_id;
+----+-------------+----------+------------+------+--------------------+--------------------+---------+-----------------------------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys      | key                | key_len | ref                         | rows | filtered | Extra |
+----+-------------+----------+------------+------+--------------------+--------------------+---------+-----------------------------+------+----------+-------+
|  1 | SIMPLE      | customer | NULL       | ALL  | PRIMARY            | NULL               | NULL    | NULL                        |  599 |   100.00 | NULL  |
|  1 | SIMPLE      | payment  | NULL       | ref  | idx_fk_customer_id | idx_fk_customer_id | 2       | sakila.customer.customer_id |   26 |   100.00 | NULL  |
+----+-------------+----------+------------+------+--------------------+--------------------+---------+-----------------------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```
可以看到在customer中是全表扫描，在表payment中customer_id作为非唯一索引扫描。

非唯一索引不管是查询单条记录或多条记录分析计划结果都是一样的是ref。
## index_merge
index_merge使用了**索引合并**得到的优化方法。

customer_id为主键，store_id为普通索引。
```sql
mysql> EXPLAIN select * from customer where customer_id=1 or store_id=1;
+----+-------------+----------+------------+-------------+-------------------------+-------------------------+---------+------+------+----------+---------------------------------------------------+
| id | select_type | table    | partitions | type        | possible_keys           | key                     | key_len | ref  | rows | filtered | Extra                                             |
+----+-------------+----------+------------+-------------+-------------------------+-------------------------+---------+------+------+----------+---------------------------------------------------+
|  1 | SIMPLE      | customer | NULL       | index_merge | PRIMARY,idx_fk_store_id | PRIMARY,idx_fk_store_id | 2,1     | NULL |  330 |   100.00 | Using union(PRIMARY,idx_fk_store_id); Using where |
+----+-------------+----------+------------+-------------+-------------------------+-------------------------+---------+------+------+----------+---------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

## range索引范围扫描
type=range.**索引范围扫描**,常见与<、<=、>、>=、between等操作符

(1)唯一索引和非唯一索引查询带双操作符
```sql
mysql> EXPLAIN select * from payment where customer_id >=300 and customer_id <= 350;
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
| id | select_type | table   | partitions | type  | possible_keys      | key                | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | payment | NULL       | range | idx_fk_customer_id | idx_fk_customer_id | 2       | NULL | 1350 |   100.00 | Using index condition |
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
(2)唯一索引带单操作符
```sql
mysql> EXPLAIN select * from payment where payment_id > 10;
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | payment | NULL       | range | PRIMARY       | PRIMARY | 2       | NULL | 8043 |   100.00 | Using where |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

(3)字符串索引也支持索引范围扫描，但只支持右%,不支持左%或左右%。
```sql
mysql> EXPLAIN select * from film where title like "ACADEMY%";
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | film  | NULL       | range | idx_title     | idx_title | 767     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

(4)还有一种比较特殊的情况，mysql会做优化，转化为范围查询,那就是同字段or查询情况。
```sql
mysql> EXPLAIN select * from address where city_id=300 or city_id=576;
+----+-------------+---------+------------+-------+----------------+----------------+---------+------+------+----------+-----------------------+
| id | select_type | table   | partitions | type  | possible_keys  | key            | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+---------+------------+-------+----------------+----------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | address | NULL       | range | idx_fk_city_id | idx_fk_city_id | 2       | NULL |    4 |   100.00 | Using index condition |
+----+-------------+---------+------------+-------+----------------+----------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
## Index索引全扫描
type=index,**索引全扫描**，MySQL遍历整个索引来查询匹配的行。
```sql
mysql> EXPLAIN select title from film;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | film  | NULL       | index | NULL          | idx_title | 767     | NULL | 1000 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## ALL全表扫描
type=All,**全表扫描**，MySQL遍历全表来找到匹配行
```sql
mysql> explain select * from film;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | film  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```

## 总结
最后有童鞋可能会说你将这么多怎么记？其实也挺好理解的，执行最快的从索引的角度大体为主键->非唯一索引->联合索引(和组合索引不同)->全索引扫描->全表扫描,从数据项的角度大体分为单条记录查询->范围查询->全表查询。
 - 主键、唯一索引的单条记录的查询 const
 - 主键、唯一索引的多条记录的查询  eq_ref
 - 普通索引单条或多条的记录查询 ref
 - 联合索引查询情况 index_merge
 - 范围查询 range
 - 全索引扫描  index
 - 全表扫描 all

最后访问的优化类型要结合实际的分析计划来定，比如说主键单条记录的查询理应type为const,但是整张表只有一条数据，那么type就应该为All。

更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)

