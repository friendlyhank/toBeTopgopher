MySQL提供了EXPLAIN或DESC命令,获取MYSQL如何执行SELECT语句得信息,包括在SELECT语句执行过程中表如何连接和连接得顺序。

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
 
接下来我们具体根据type进行讲解。
## TYPE
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200520180848914.png)
从左到右，性能由差变好。

### ALL全表扫描
type=All,全表扫描，MySQL遍历全表来找到匹配行
```sql
mysql> explain select * from film;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | film  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```

### Index索引全扫描
type=index,索引全扫描，MySQL遍历整个索引来查询匹配的行。
```sql
mysql> EXPLAIN select title from film;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | film  | NULL       | index | NULL          | idx_title | 767     | NULL | 1000 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## range索引范围扫描
(1)
type=range.索引范围扫描,常见与<、<=、>、>=、between等操作符：
```sql
mysql> EXPLAIN select * from payment where customer_id >=300 and customer_id <= 350;
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
| id | select_type | table   | partitions | type  | possible_keys      | key                | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | payment | NULL       | range | idx_fk_customer_id | idx_fk_customer_id | 2       | NULL | 1350 |   100.00 | Using index condition |
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

注意单个操作符执行的不是索引的范围扫描:
```sql
mysql> EXPLAIN select * from payment where customer_id >=300;
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys      | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | payment | NULL       | ALL  | idx_fk_customer_id | NULL | NULL    | NULL | 16086 |    49.20 | Using where |
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

(2)字符串索引也支持索引范围扫描，只是只支持右%,不支持左%或左右%。
```sql
mysql> EXPLAIN select * from film where title like "ACADEMY%";
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | film  | NULL       | range | idx_title     | idx_title | 767     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

注意左%或左右%不支持索引范围扫描：
```sql
mysql> EXPLAIN select * from film where title like "%ACADEMY%";
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | film  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |    11.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## index_merge
index_merge使用了索引合并得优化方法。customer_id为主键，store_id为普通索引。
```sql
mysql> EXPLAIN select * from customer where customer_id=1 or store_id=1;
+----+-------------+----------+------------+-------------+-------------------------+-------------------------+---------+------+------+----------+---------------------------------------------------+
| id | select_type | table    | partitions | type        | possible_keys           | key                     | key_len | ref  | rows | filtered | Extra                                             |
+----+-------------+----------+------------+-------------+-------------------------+-------------------------+---------+------+------+----------+---------------------------------------------------+
|  1 | SIMPLE      | customer | NULL       | index_merge | PRIMARY,idx_fk_store_id | PRIMARY,idx_fk_store_id | 2,1     | NULL |  330 |   100.00 | Using union(PRIMARY,idx_fk_store_id); Using where |
+----+-------------+----------+------------+-------------+-------------------------+-------------------------+---------+------+------+----------+---------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

## ref非唯一索引扫描
(1)
type=ref,使用非唯一索引扫描,返回匹配某个单独值得记录行。
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

## eq_ref
type=eq_ref,类似于ref,区别在使用索引是唯一索引，简单来说，就是多表连接中使用primary key或者unique index作为关联条件。
主键相连得例子：
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

## const/system
type=const/system,单表中最多一个匹配行，查询起来非常迅速。const是根据primary key或者唯一索引unique index进行查询,system表示系统只有一条数据，这是一个特殊得const类型。
```sql
mysql> EXPLAIN select * from film where film_id=1;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | film  | NULL       | const | PRIMARY       | PRIMARY | 2       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

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

