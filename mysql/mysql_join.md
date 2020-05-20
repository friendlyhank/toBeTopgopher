> 数据库中join又称连接，用于联合多表的查询。

我们会以两张表做例子customer：
```sql
mysql> select * from customer;
+------------+-----------+
| customerid | name      |
+------------+-----------+
|          1 | Joe       |
|          2 | Christian |
|          3 | Zero      |
|          4 | Karl      |
+------------+-----------+
4 rows in set (0.00 sec)
```
```sql
mysql> select * from `order`;
+---------+------------+
| orderid | customerid |
+---------+------------+
|   10001 |          1 |
|   10002 |          2 |
|   10003 |          3 |
|   10005 |          5 |
+---------+------------+
4 rows in set (0.00 sec)
```

## inner join内连接
inner join或者两个表中字段匹配关系的记录。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200519154427302.png)
```sql
select * from customer INNER JOIN `order` on customer.customerid=`order`.customerid;
+------------+-----------+---------+------------+
| customerid | name      | orderid | customerid |
+------------+-----------+---------+------------+
|          1 | Joe       |   10001 |          1 |
|          2 | Christian |   10002 |          2 |
|          3 | Zero      |   10003 |          3 |
+------------+-----------+---------+------------+
3 rows in set (0.00 sec)
```

## left join左连接
获取左边所有记录，即使右表没有对应的匹配的记录。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200519154436240.png)
```sql
mysql> select * from customer left JOIN `order` on customer.customerid=`order`.customerid;
+------------+-----------+---------+------------+
| customerid | name      | orderid | customerid |
+------------+-----------+---------+------------+
|          1 | Joe       |   10001 |          1 |
|          2 | Christian |   10002 |          2 |
|          3 | Zero      |   10003 |          3 |
|          4 | Karl      |    NULL |       NULL |
+------------+-----------+---------+------------+
4 rows in set (0.00 sec)
```

## right join右连接
与left join相反，用于获取右边所有记录，即使左边没有对应匹配的记录。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200519154445999.png)
```sql
mysql> select * from customer right JOIN `order` on customer.customerid=`order`.customerid;
+------------+-----------+---------+------------+
| customerid | name      | orderid | customerid |
+------------+-----------+---------+------------+
|          1 | Joe       |   10001 |          1 |
|          2 | Christian |   10002 |          2 |
|          3 | Zero      |   10003 |          3 |
|       NULL | NULL      |   10005 |          5 |
+------------+-----------+---------+------------+
4 rows in set (0.00 sec)
```
## LEFT JOIN EXCLUDING INNER JOIN (左连接 - 内连接)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200519154559243.png)
```sql
mysql> select * from customer LEFT JOIN `order` on customer.customerid=`order`.customerid where `order`.customerid IS NULL;
+------------+------+---------+------------+
| customerid | name | orderid | customerid |
+------------+------+---------+------------+
|          4 | Karl |    NULL |       NULL |
+------------+------+---------+------------+
1 row in set (0.00 sec)
```

## RIGHT JOIN EXCLUDING INNER JOIN (右连接 - 内连接)

```sql
mysql> select * from customer RIGHT JOIN `order` on customer.customerid=`order`.customerid where `customer`.customerid IS NULL;
+------------+------+---------+------------+
| customerid | name | orderid | customerid |
+------------+------+---------+------------+
|       NULL | NULL |   10005 |          5 |
+------------+------+---------+------------+
1 row in set (0.00 sec)
```

## CROSS JOIN笛卡尔积
左边的每一条记录和右边每一条记录组成数据对。
```sql
mysql> select * from customer CROSS JOIN `order`;
+------------+-----------+---------+------------+
| customerid | name      | orderid | customerid |
+------------+-----------+---------+------------+
|          1 | Joe       |   10001 |          1 |
|          2 | Christian |   10001 |          1 |
|          3 | Zero      |   10001 |          1 |
|          4 | Karl      |   10001 |          1 |
|          1 | Joe       |   10002 |          2 |
|          2 | Christian |   10002 |          2 |
|          3 | Zero      |   10002 |          2 |
|          4 | Karl      |   10002 |          2 |
|          1 | Joe       |   10003 |          3 |
|          2 | Christian |   10003 |          3 |
|          3 | Zero      |   10003 |          3 |
|          4 | Karl      |   10003 |          3 |
|          1 | Joe       |   10005 |          5 |
|          2 | Christian |   10005 |          5 |
|          3 | Zero      |   10005 |          5 |
|          4 | Karl      |   10005 |          5 |
|          1 | Joe       |   10004 |          1 |
|          2 | Christian |   10004 |          1 |
|          3 | Zero      |   10004 |          1 |
|          4 | Karl      |   10004 |          1 |
+------------+-----------+---------+------------+
20 rows in set (0.00 sec)
```

参考：[https://blog.csdn.net/zhengsy_/article/details/90733864](https://blog.csdn.net/zhengsy_/article/details/90733864)

更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)





