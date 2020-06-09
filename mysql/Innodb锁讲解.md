## 概述
上一章我们讲到，MyISAM和MEMORY存储引擎采用的是表级锁；BDB存储引擎采用的是页面锁，但也支持表级锁；InnoDB存储引擎即支持行级锁，也支持表级锁。

## 查询锁的状态
可以通过检查InnoDB_row_lock状态变量来分析系统上行锁的争夺情况：
```sql
mysql> show  status like 'innodb_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_row_lock_current_waits | 0     |
| Innodb_row_lock_time          | 0     |
| Innodb_row_lock_time_avg      | 0     |
| Innodb_row_lock_time_max      | 0     |
| Innodb_row_lock_waits         | 0     |
+-------------------------------+-------+
5 rows in set (0.01 sec)
```
如果发现锁争用比较严重，如InnoDB_row_lock_waits和InnoDB_row_lock_time_avg的值比较高

可以通过查询information_schema数据库的表了解锁等待情况:
```bash
mysql> use information_schema;
Database changed
mysql> select * from innodb_locks;
+----------------+-------------+-----------+-----------+-------------------+------------+------------+-----------+----------+-----------+
| lock_id        | lock_trx_id | lock_mode | lock_type | lock_table        | lock_index | lock_space | lock_page | lock_rec | lock_data |
+----------------+-------------+-----------+-----------+-------------------+------------+------------+-----------+----------+-----------+
| 1148205:30:3:2 | 1148205     | X         | RECORD    | `miaosha`.`goods` | PRIMARY    |         30 |         3 |        2 | 100000    |
| 1148203:30:3:2 | 1148203     | X         | RECORD    | `miaosha`.`goods` | PRIMARY    |         30 |         3 |        2 | 100000    |
+----------------+-------------+-----------+-----------+-------------------+------------+------------+-----------+----------+-----------+
2 rows in set, 1 warning (0.00 sec)

mysql> select * from innodb_lock_waits;
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 1148212           | 1148212:30:3:3    | 1148213         | 1148213:30:3:3   |
+-------------------+-------------------+-----------------+------------------+
1 row in set, 1 warning (0.00 sec)
```

## 共享锁和排它锁
InnoDB实现了两种类型行锁：
 - 共享锁(S)：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁。
 - 排他锁(X): 允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享锁和排他锁。

**对于UPDATE、DELETE和INSERT语句InnoDB会自动给涉及数据集加排它锁(X);对于普通SELECT语句，InnoDB不会加任何锁**;

### 行锁的具体使用

事务可以通过一下语句显示给记录集加共享锁或排他锁：
 - 共享锁(S):
```sql
SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE
```
 - 排它锁(X)：
```sql
SELECT * FROM table_name WHERE ... LOCK for update
```

**共享锁**
用SELECT ...IN SHARE MODE获得共享锁，主要用在需要数据依存关系时来确认某行记录是否存在，并确保没有人对这个记录进行UPDATE或者DELETE操作。但是如果当前事务也需要对该记录进行更新操作，则有可能造成死锁，对于锁定记录后进行更新操作，应该使用SELECT...FOR UPDATE方式获得排它锁。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200602081029806.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
前面提到update操作，会自动的加排它锁(X)，在session2获得共享锁之后，session1的update操作想获得排它锁而进入阻塞状态，这时候session2如果继续对进入阻塞状态的记录去占用锁就会导致死锁，注意在同session中获取共享锁和排它锁不会进入阻塞状态。

**排它锁**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200604141916791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
MySQL的**行锁是针对索引加的锁**，不是针对记录加的锁，不管是单索引还是多索引，在检查锁是否冲突的时候，要看它的索引键是否出现冲突。
 
 (1)在不通过索引条件查询时，InnoDB会锁定表中所有记录。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200604183356201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
(2)索引和非索引的情况
如下图所示store_id为索引，first_name为非索引，session1和session2为两条索引store_id=1的不同记录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200605091204360.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
(3)多索引的情况
如图customer_id和rental_id分别对应两个索引,MySQL行锁会给两个索引都加锁，当两个索引对应不同的记录的时候，不会有锁冲突的情况，MySQL行锁会给两个索引都加上锁。

 - 不同索引不同记录的情况
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060509523785.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 - 不同索引同记录的情况
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060517574527.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
(4)复合索引的情况
我们知道在复合索引中,假设col1+col2+col3是复合的索引字段,根据最左匹配原则查询的时候col1,col1+col2,col1+col3,col1+col2+col3是走索引的，而col2,col3,col2+col3是不走索引的，锁也是这样的。
 - 走索引的情况,只举例col1+col2+col3
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200605102826617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 - 不走索引的情况col2,会导致锁表
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200605104000466.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

**在MySQL中，是否使用索引是通过执行计划Explain来决定的，如果执行计划没有走索引，InnoDB就会对所有记录加锁，反之则是行锁。**

## 意向锁
 另外，为了允许行锁和表锁共存，实现多粒度机制，InnoDB还有两种内部使用的意向锁(Intention Locks)，这种意向锁都是表锁。
 
上述锁模式的兼容情况：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200605182002759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
意向锁锁定协议如下:
 - 在事务想要获取某行上的共享锁(S)之前，他首先必须获取IS意向锁或更高级的锁。
 - 在事务想要获取某行上的排它锁(X)之前，他首先必须获取IX意向锁或更高级的锁。

如果一个事务请求的锁模式与当前的锁兼容，InnoDB就将请求的锁授予该事务；反之，如果两者不兼容，该事务就要等待锁释放，意向锁是InnoDB自动加的，不需要用户干预。
意向锁的主要目的是表明有人正在锁定表中的行，或者打算锁定表中的行，试想一下如果没有意向锁，假设现在要获取一个表锁，是不是要一行行检查是否有行锁才可以？
## 行锁实现方式：
InnoDB行锁是通过给索引上的索引项加锁来实现的，如果没有索引，InnoDB将通过隐藏的聚簇索引来对记录加锁。InnoDB行锁分为3种情形。
### Record Lock
记录锁是对索引记录的锁定。
###  Grap lock
间隙锁是对索引记录之间间隙的锁定，或者是对第一条记录或最后一个索引记录之前的间隙的锁定。
间隙锁是性能和并发性之间权衡的一部分，并且在某些事务隔离级别而非其他级别中使用。

对于使用唯一索引来锁定的语句，不需要间隙锁定，对于非唯一和没有索引的语句，需要间隙锁定。
### Next-key lock
Next-key lock:前两种的组合，对记录及其前面的间隙加锁。

在范围查询条件中，使用共享锁或者排它锁,在键值查询条件不存在的记录就叫做"间隙"(GAP),如果只对不存在“间隙”加锁，就是Grap lock锁，对记录和"间隙"的都加锁，就是Next-key 锁。

举例来说，假设有一张user表
```bash
CREATE TABLE `user` (
  `id` int(1) NOT NULL AUTO_INCREMENT,
  `name` varchar(8) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

INSERT INTO `user` VALUES ('1', '小罗');
INSERT INTO `user` VALUES ('7', '小明');
INSERT INTO `user` VALUES ('11', '小红');
INSERT INTO `user` VALUES ('18', '老紫');
INSERT INTO `user` VALUES ('20', '紫涵');
```
也就是id得区间范围是
[1,7)
[7,11)
[11,18)
[18,20]

我们来根据具体得例子来具体分析：
对记录数据加锁：
 - 相等
```sql
select * from `user` where id =11 FOR UPDATE;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200608212658391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
可以看到对具体某条存在得记录加锁，只会对这条记录加记录锁。

 - BETWEEN范围
```sql
select * from `user` where id BETWEEN 7 and 11 FOR UPDATE;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200608212951562.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
记录7,11会被加记录锁,(7,11)(12,18)会被加间隙锁。
 - 范围
```sql
SELECT * FROM `user` WHERE `id` >= 7 FOR UPDATE;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200608222752916.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
记录7、11、18会被加记录所,(7,11)(11,18)(18,+∞)会被加间隙锁。

对间隙加锁：
 - 相等
```sql
select * from `user` where id =14 FOR UPDATE;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200608223317526.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
对(11,18)的间隙加锁。

 - BETWEEN范围
```sql
select * from `user` where id BETWEEN 14 and 16 FOR UPDATE;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060822334819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
对(11,18)的间隙加锁。

 - 范围
```sql
SELECT * FROM `user` WHERE `id` >= 15 FOR UPDATE;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200609094627400.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
对18加记录锁,对(11,18)(18,+∞)的间隙加锁。

上述示例是在mysql默认隔离级别REPEATABLE-READ下，实际上，在不同的隔离级别下，InnoDB处理SQL时采用的一致性读策略和需要的锁是不同的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200609091312703.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

## 问题
锁是多久就会自动释放?
答：innodb_lock_wait_timeout默认是50秒


