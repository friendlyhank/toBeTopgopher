## show status查看服务器状态信息
mysql提供show status命令可以查看服务器的状态信息,故可以用这个命令了解各个SQL的执行频率。
```sql
SHOW [GLOBAL | SESSION] STATUS
    [LIKE 'pattern' | WHERE expr]
```
从命令可以看出show status分别可以从全局global和当前session去查询服务器的状态信息，不填的话默认就是session,同时这个命令也可以通过like匹配相关的变量名称。
```sql
show global status like "Com_%";
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| Com_begin                 | 391646  |
| Com_rollback              | 1076    |
| Com_commit                | 390490  |
+---------------------------+---------+
```
对于事务型应用，通过Com_begin、Com_rollback、Com_commit可以了解事务的开启、回滚、提交的情况，如果事务开启未关闭容易造成死锁，如果回滚操作比较多，则可能程序编写有异常。
```sql
show global status like "Com_%";
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| Com_select                | 4111471 |
| Com_update                | 928424  |
| Com_insert                | 572114  |
| Com_delete                | 566     |
+---------------------------+---------+
```
对于些crud的语句操作，可以分析当前数据库应用到底是以插入为主还是以查询为主，来调整响应的策略。
```sql
mysql> show GLOBAL status like "Slow_%";
+---------------------+--------+
| Variable_name       | Value  |
+---------------------+--------+
| Slow_queries        | 331654 |
+---------------------+--------+
2 rows in set
```
Slow_queries慢查询的次数


官方参考:[show-status](https://dev.mysql.com/doc/refman/5.7/en/show-status.html)

更多系统学习欢迎关注github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)





