为了方便演示，大家可以下载官方[sakila示例数据库文件](https://dev.mysql.com/doc/index-other.html)

## 使用索引典型场景
### 范围查询双操作符
```sql
mysql> EXPLAIN select * from payment where customer_id >=300 and customer_id <= 350;
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
| id | select_type | table   | partitions | type  | possible_keys      | key                | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | payment | NULL       | range | idx_fk_customer_id | idx_fk_customer_id | 2       | NULL | 1350 |   100.00 | Using index condition |
+----+-------------+---------+------------+-------+--------------------+--------------------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

### 匹配最左前缀
索引是从索引的最左边列进行查找的，比如col1+col2+col3字段上的索引能被包含col1、col1+col2、col1+col2+col3的等值查询利用到，可是不会被col2、col2+col3、col3的等值查询利用到;以支付表payment为例，如果查询条件中仅包含索引的第一列支付日志payment_date和索引的第三列更新时间last_update的时候，可以使用到索引：
```sql
mysql> EXPLAIN select * from payment where payment_date='2006-02-14 15:16:03' and last_update='2006-02-15 22:12:32';
+----+-------------+---------+------------+------+------------------+------------------+---------+-------+------+----------+-----------------------+
| id | select_type | table   | partitions | type | possible_keys    | key              | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+---------+------------+------+------------------+------------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | payment | NULL       | ref  | idx_payment_date | idx_payment_date | 5       | const |  182 |    10.00 | Using index condition |
+----+-------------+---------+------------+------+------------------+------------------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

当使用符合索引idx_payment_date的第二列支付金额amount和第三列更新时间last_update进行查询时，不会利用到索引：
```sql
mysql> EXPLAIN select * from payment where amount=3.98 and last_update='2006-02-15 22:12:32';
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | payment | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 16086 |     1.00 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

### 匹配列前缀
```sql
mysql> EXPLAIN select * from film where title like "ACADEMY%";
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | film  | NULL       | range | idx_title     | idx_title | 767     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

## 不使用索引的典型场景
 ### 开头%或首位%不支持索引范围扫描：
```sql
mysql> EXPLAIN select * from film where title like "%ACADEMY";
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | film  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |    11.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

### 非唯一索引单操作符不支持索引范围扫描:
```sql
mysql> EXPLAIN select * from payment where customer_id >=300;
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys      | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | payment | NULL       | ALL  | idx_fk_customer_id | NULL | NULL    | NULL | 16086 |    49.20 | Using where |
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

### 数据类型出现隐式转换
在where条件中把字符串常量值用引号引起来，否则即便这个列上有索引，MySQL也不会用到,因为MSYQL默认把输入的常量进行转换后才进行检索。如设置了idx_actor_last_name索引的字符串类型last_name,如果查询是一个数值类型，则不走索引。
```sql
mysql> EXPLAIN select * from actor where last_name=1;
+----+-------------+-------+------------+------+---------------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys       | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | actor | NULL       | ALL  | idx_actor_last_name | NULL | NULL    | NULL |  202 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------------+------+---------+------+------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)
```

### 用or隔开的条件
如果or前条件中的列有索引，而后面的列没有索引，那么涉及的索引都不会被用到。
```sql
mysql> EXPLAIN select *  from payment where customer_id=203 or amount=3.96;
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys      | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | payment | NULL       | ALL  | idx_fk_customer_id | NULL | NULL    | NULL | 16086 |    10.15 | Using where |
+----+-------------+---------+------------+------+--------------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)
