## 分布式事务原理
在MySQL中，使用分布式事务的应用程序及一个或多个资源管理器和一个事务管理器。
 - 资源管理器(RM)用于提供通向事务资源的途径。数据库服务器是一种资源管理器。该管理器必须可以提交或回滚由RM管理的事务。
 - 事务管理器( TM)用于协调作为一个分布式事务一部分的事务。TM与管理每个事务的RMs进行通信。在一个分布式事务中，各个单个事务均是分布式事务的“分支事务”。分布式事务和各分支通过一种命名方法进行标识。
	MySQL执行XA MySQL时，MySQL服务器相当于一个用于管理分布式事务中XA事务的资源管理器。与MySQL服务器连接的客户端相当于事务管理器。

**两阶段提交 (Two-Phase Commit, 简称2PC):**
 - 在第一阶段,所有分支被预备好。即他们被TM告知要准备提交。通常，这意味着用于管理分支的每个RM会记录对于被稳定保存的分支的行动。分支指示是否可以这么做。这些结果被用于第二阶段。
 - 在第二阶段，TM告知RMS是否要提交或回滚。如果在预备分支时，所有的分支指示他们能够提交，则所有分支被告知要提交。如果在预备时,有任何分支指示它将不能提交，则所有分支被告知回滚。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052717174120.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
## 分布式事务语法
```sql
xa {start|begin} xid [join|resume] //开启一个事务，并将事务置于ACTIVE状态。
xa end xid [suspend [for migrate]] //将事务置于IDLE状态,表示事务内的SQL操作完成
xa prepare xid //将事务进入prepare状态，就是两阶段的第一个提交阶段,并会把xa start到xa end之间操作记录到binlog中
xa rollback xid  //第二阶段分支事务的回滚
xa commit xid //第二阶段分支事务的提交
xa recover //返回当前数据库中处于PREPARE状态的分支事务的详情信息
```
**事务状态变迁图：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200527171820251.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

MySql的数据库存储引擎InnoDB的事务特性能够保证存储引擎级别实现ACID，而分布式事务让存储引擎级别的事务扩展到数据库层面，甚至扩展到多个数据库之间，这是通过两阶段提交协议来实现的。

## 分布式案例
```sql
#session1
mysql> xa start 'test','sakila';
Query OK, 0 rows affected (0.00 sec)
```

```sql
#session2
mysql> xa start 'test','saklitest';
Query OK, 0 rows affected (0.00 sec)
```

```sql
#session1
mysql> insert into actor (actor_id,first_name,last_name) values (303,'smid','smid');
Query OK, 1 row affected (0.02 sec)

#对分支事务1进行第一阶段提交
mysql> xa end 'test','sakila';
Query OK, 0 rows affected (0.00 sec)

mysql> xa prepare 'test','sakila';
Query OK, 0 rows affected (0.01 sec)
```

```sql
#session2
mysql> update film_actor set last_update=now() where actor_id=178;
Query OK, 23 rows affected (0.00 sec)
Rows matched: 23  Changed: 23  Warnings: 0
#对分支事务2进行第二阶段提交
mysql> xa end 'test','saklitest';
Query OK, 0 rows affected (0.00 sec)

mysql> xa prepare 'test','saklitest';
Query OK, 0 rows affected (0.01 sec)
```

用xa recover命令查看session1当前分支事务状态
```sql
#session1
mysql> xa recover;
+----------+--------------+--------------+---------------+
| formatID | gtrid_length | bqual_length | data          |
+----------+--------------+--------------+---------------+
|        1 |            4 |            6 | testsakila    |
|        1 |            4 |            9 | testsaklitest |
+----------+--------------+--------------+---------------+
2 rows in set (0.00 sec)
```

用xa recover命令查看session2当前分支事务状态。
```sql
#session2
mysql> xa recover;
+----------+--------------+--------------+---------------+
| formatID | gtrid_length | bqual_length | data          |
+----------+--------------+--------------+---------------+
|        1 |            4 |            6 | testsakila    |
|        1 |            4 |            9 | testsaklitest |
+----------+--------------+--------------+---------------+
2 rows in set (0.00 sec)
```

第二阶段进入提交阶段，如果遇到问题则进行回滚，反之，则提交分布式事务，保证事务的正确性。
```sql
#session1
mysql> xa commit 'test','sakila';
Query OK, 0 rows affected (0.01 sec)
```
sesion1执行xa commit之后已经可以查询到插入数据,这时候session2还没执行xa commit。
```sql
#session1
mysql> select * from actor where first_name='Simon';
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      301 | Simon      | Tom       | 2020-05-21 21:53:46 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)
```

```sql
#session2
mysql> xa commit 'test','saklitest';
Query OK, 0 rows affected (0.01 sec)
```


更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)
