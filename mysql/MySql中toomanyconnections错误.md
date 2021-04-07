有时候在使用mysql过程中会出现"too many connections"的错误，顾名思义是因为连接数过多造成的，造成连接数过多的原因主要有：
 - **系统硬件等限制**：根据官方说明,连接数受到线程库质量、RAM的可用性，响应时间,可用的文件描述符数量等影响，这种方式的解决方式为增加服务器或提高服务器的性能。
 - **连接的释放**：这个很好理解，连接如果没有被正常释放，也可能导致连接数过多。
 - **最大连接数设置过低**

前面两种情况不做过多讲解，重点来讲一下如何合理的设置最大连接数。
## max_connections
查看最大连接数的命令为：
```bash
mysql> show variables like '%max_connections%';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
1 row in set (0.00 sec)
```
最大连接数默认值为151，最大值可设置为100000,实际上mysql允许max_connections+1个客户端连接，那是因为需要保留额外的连接供super特权的账户使用，比如特权账户在连接过多时候可以使用SHOW PROCESSLIST诊断问题。

在每个连接工作负载较低的情况下，一个拥有几GB内存Linux或Solaris可以同时支撑500至1000的连接,最多可支持10000个连接。


其次还有几个配置参数是我们不容忽视的：
**Max_used_connections**服务器响应的最大连接数
```bash
mysql> show global status like 'Max_used_connections';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Max_used_connections |   2   |
+----------------------+-------+
1 row in set (0.02 sec)
```
我们可以通过服务器响应的最大连接数知道平时服务器连接数的峰值能去到多少，然后根据这个数值去设置合理的连接数,一般来说服务器响应的最大连接数值占最大连接数值占比应大于10%且不应该超过85%。例如：
```bash
Max_used_connections / max_connections * 100% = 2 / 151 *100%  ≈ 1%
```
可以看到占比低于10%，说明最大连接数设置过大(这个结果只是测试，计算要根据实际情况),同理当占比超过85%说明预留的连接数较小，遇到突发情况也可能超出限制。

**Threads**查看当前线程数
```bash
mysql> show status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 6     | //线程缓存中的连接数
| Threads_connected | 3     | //当前打开的连接数
| Threads_created   | 792   | //被创建的线程数
| Threads_running   | 1     | //未休眠的线程数
+-------------------+-------+
4 rows in set (0.00 sec)
```
这个参数可以查看到当前线程连接情况。

## 设置max_connections
设置max_connections连接数的方式有两种:

 ### 通过命令方式设置连接数
 
 ```bash
mysql> set GLOBAL max_connections=1500;
Query OK, 0 rows affected (0.00 sec)
```
当mysql发生重启的时候这种方式设置会使最大连接数恢复为默认值。

### 修改配置文件
打开配置,在配置中添加max_connections=1000,重启mysql即可生效。
```bash
//打开配置文件
vi /etc/my.cnf

//配置中添加
max_connections=1000

//重启mysql 
service mysqld restart
```







