## 命令开启慢查询
查看是否开启慢查询
slow_query_log慢查询开启状态,默认OFF
slow_query_log_file慢查询日志文件存放的路径
```sql
mysql> show variables like 'slow_query%';
+---------------------+-----------------------------------+
| Variable_name       | Value                             |
+---------------------+-----------------------------------+
| slow_query_log      | OFF                               |
| slow_query_log_file | /var/lib/mysql/localhost-slow.log |
+---------------------+-----------------------------------+
2 rows in set (0.00 sec)
```

查询慢查询设置的时间
long_query_time 慢查询默认设置的时间,默认为10秒
```sql
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.00 sec)
```

设置开启慢查询

```sql
set global slow_query_log='ON';
Query OK, 0 rows affected (0.00 sec)
```

更改路径
```sql
set global slow_query_log_file='/var/lib/mysql/mysql-slow.log';
Query OK, 0 rows affected (0.00 sec)
```

更改时间，将慢查询的时间改为1秒
```sql
mysql> set global long_query_time=1;
Query OK, 0 rows affected (0.00 sec)
```

测试一下慢查询
```sql
select sleep(2)
```

查询一下日志
```bash
cat /var/lib/mysql/mysql-slow.log
# Time: 200506 15:15:41
# User@Host: root[root] @ localhost []  Id:    11
# Query_time: 2.000399  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
SET timestamp=1588749341;
select sleep(2);
```

## 配置设置慢查询
linux：配置文件为etc/my.cnf
```sql
[mysqld]  
long_query_time=2  

#5.5以前版本配置如下选项  
log-slow-queries="mysql_slow_query.log"  
#5.5及以上版本配置如下选项  
slow-query-log=On  
slow_query_log_file="mysql_slow_query.log"  

log-query-not-using-indexes
```

更改配置之后重启生效:
```powershell
service mysql restart
```



