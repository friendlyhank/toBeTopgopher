## 概述
在分布式系统中为了解决单点问题，通常会把数据复制多个副本部署到其他机器，满足故障恢复和负载均衡等需求。redis也是如此，为我们提供了复制功能。

参与复制的redis实例分为主节点和从节点，默认情况下,redis都是主节点。每个主节点可以有多个从节点，每个从节点只能有一个主节点，复制是单向的，只能由主节点复制到子节点。
比方说现在有两台机192.168.85.110主节点，192.168.85.111是从节点,要建立如下主从关系。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401164432471.png)
## 前置-安全性校验
在开启主从复制之前，redis配置默认有保护模式配置，需要设置一下，否则会导致开启主从复制失败。

方式一修改保护模式：
redis默认只允许本地客户端进行连接,所以在设置主从复制的时候，需要关闭主节点的保护,在配置中修改protected-mode为no
```powershell
protected-mode yes
```
方式二设置口令（推荐）：
修改主节点配置，设置密码auth
```powershell
requirepass 123456
```
修改从节点的配置，设置主节点的密码
```powershell
masterauth 123456
```
设置口令或者关闭保护模式之后就可以设置主从复制了。
## 主从复制的命令：
 - 在配置中直接添加:
```powershell
slaveof 192.168.85.110 6379
```
- 在redis-server启动命令时候--slaveof host port
```powershell
./src/redis-server --slaveof 192.168.85.110 6379 &
```
 - 直接设置命令 slaveof host port
```powershell
slaveof 192.168.85.110 6379
```
如果主节点有设置密码，记得要在从节点的配置文件中设置主节点的密码,并且启动时候让配置生效。

设置完成之后，可以对主从节点进行测试，尝试从主节点写入，看有无复制到从节点：
110主节点节点:
```powershell
127.0.0.1:6379> set say helloworld
OK
127.0.0.1:6379> get say
"helloworld"
```
111从节点：
```powershell
127.0.0.1:6379> get say
"helloworld"
```

同时也可以通过info replication查看复制相关的状态信息
```powershell
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=192.168.85.111,port=6379,state=online,offset=899,lag=1
master_replid:5179c51dbd9b90f3fbdeb7cf9083d71bfa5c8509
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:899
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:899
```

## 切主
slaveof还可以实现切主操作，既是可以从节点对主节点的复制，变成从节点对另外一个主节点的复制。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401170955955.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
切主操作流程如下：

 1. 断开与旧主节点复制关系。
 2. 与新主节点建立复制关系。
 3. 删除从节点当前所有数据。
 4. 对新主节点进行复制操作。
 
## 去除主从关系
```powershell
slaveof no one
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401164738735.png)
```powershell
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:0
master_replid:5179c51dbd9b90f3fbdeb7cf9083d71bfa5c8509
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2917
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2917
```

断开复制的主要流程：
 1. 断开与主节点的复制关系
 2. 从节点晋升为主节点

 从节点断开复制后不回抛弃原有的数据，只是无法再获取主节点的变化信息。


## 注意点
 - 设置主从或切主之后，从节点数据都会被删除，所以要特别小心。
 - 设置主从复制关系之后，主节点要注意设置好持久化，防止断电重启之后，主节点没有数据，从节点由于复制主节点，导致从节点数据也被清空的问题。
 - 一般情况下，从节点默认的配置replica-read-only为只读模式，由于复制是单向的，只能是主节点到从节点，为了保持主从数据的一致性，应不要随意修改该配置。


