> 当遇到单机内存、并发、流量等瓶颈时，可以采用Cluster架构方案达到负载均衡的目的。

学习redis集群要从数据分布、搭建集群、节点通信、集群伸缩、请求路由、故障转移、集群运维等方面入手。
## redis数据分区
Redis Cluser采用虚拟槽分区，所有的键根绝哈希函数映射到0~16383整数槽内，每一个节点负责维护一部分槽及槽的映射的键值数据。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200426195433103.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200426195446362.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)


## 集群功能限制
Redis集群相对单机在功能上存在限制，需要开发人员提前了解，在使用时做好规避。限制如下：
 1. key批量操作支持有限。如mset、mget,目前只支持具有相同slot值的key执行批量操作。对于映射为不同slot值得key由于执行mget、mget等操作可能存在于多个节点上因此不被支持。
 2. key事务操作支持有限。同理只支持多key在同一个节点上得事务操作，当多个key分布在不同节点上时无法使用事务功能。
 3. key作为数据分区得最小粒度，因此不能被一个大的键值对象如hash、list等映射到不同得节点。
 4. 不支持多数据库空间。单机下得Redis可以支持16个数据库，集群模式下只能使用一个数据库空间，即db0。
 5. 复制结构只支持一层，从节点只能复制主节点，不支持嵌套树状复制结构。

## 搭建集群
 1. 准备节点。
 2. 节点握手
 3. 分配槽

### 准备节点
Redis集群一般由多个节点组成，节点数量至少为6个才能保证组成完整高可用得集群。每个节点需要配置cluster-enabled yes,让redis运行在集群模式下。
```
#节点端口
port 6379
# 开启集群模式
cluster-enabled yes
# 节点超时时间， 单位毫秒
cluster-node-timeout 15000
# 集群内部配置文件
cluster-config-file "nodes-6379.conf"
```
第一次启动得时候如果没有集群配置文件，它会自动创建一份，文件名称采用cluster-config-file参数控制，建议采用node-{port}.conf格式定义，使用端口号区分不同节点，防止同一个机器下多个节点彼此覆盖，造成集群信息异常。如果启动时存在集群配置文件，节点会使用配置文件内存初始化集群信息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200426202154506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
集群模式的Redis除了原有的配置文件之外又加了一份集群配置文件。当集群内节点信息发生变化， 如添加节点、 节点下线、 故障转移等。 节点会自动保存集群状态到配置文件中。 需要注意的是， Redis自动维护集群配置文件， 不要手动修改， 防止节点重启时产生集群信息错乱。

### 节点握手
设置完成之后就要设置节点握手，节点握手是一批运行在集群模式下得节点通过Gossip协议彼此通信，达到感知对方的过程。节点握手时集群彼此通信得第一步，由客户端发起命令：cluster meet {ip} {port}
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200427075143198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
以6379为主节点，打开主节点客户端，执行命令cluster meet 127.0.0.1 6380,让节点6379和6380节点进行握手通信。cluster meet命令时一个异步命令，执行之后立刻返回。内部发起与目标节点进行握手通信。
 1. 节点6379本地创建6380节点信息对象，并发送meet消息。
 2. 节点6380接收到meet消息后，保存6379节点信息并回复pong消息。
 3. 之后节点6379和6380彼此定期通过ping/pong消息进行正常的节点通信。

输出集群节点信息，来查看集群信息以及判断是否设置成功：
```
cluster nodes
```

节点建立握手之后集群还不能正常工作，这时集群处于下线状态，所有得数据读写都被禁止。
```
 cluster info
```
```
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:0
cluster_current_epoch:5
cluster_my_epoch:1
cluster_stats_messages_sent:234
cluster_stats_messages_received:234
```
可以看到,被分配的槽(cluster_slots_assigned)是0，由于目前所有的槽没有分配到节点，因此集群无法完成槽到节点得映射。只有当16384个槽全部分配给节点后，集群才进入在线状态。

### 分配槽
Redis集群把所有的数据映射到16384个槽中。每个key会映射为一个固定的槽，只有当节点分配了槽，才能响应和这些槽关联得键命令。通过cluster addslots命令为节点分配槽。注意下面...是省略意思，要一个个加。
```
redis-cli -h 127.0.0.1 -p 6379 cluster addslots {0...5461}
redis-cli -h 127.0.0.1 -p 6380 cluster addslots {5462...10922}
redis-cli -h 127.0.0.1 -p 6381 cluster addslots {10923...16383}
```

再次执行cluster nodes命令可以看到节点和槽的分配关系：
```
f6340f10b3518f779b87451495c5c2375fbf56b1 127.0.0.1:6383 master - 0 1587952045136 0 connected
d67db857f8d6549b74c8e4b5b2cc8a78dfa918d3 127.0.0.1:6380 master - 0 1587952043129 4 connected 6-11
68bb0474810018d588582bd5a782ad0abcccdd30 127.0.0.1:6381 master - 0 1587952044132 2 connected 12-17
1dfe8f21f7379bbcfd9fc233fbff73e6f5fc8b6d 127.0.0.1:6382 master - 0 1587952042121 3 connected
bc4b2d895a8b330252b1bba6d477040a5c269f14 127.0.0.1:6384 master - 0 1587952046142 5 connected
94756f9966f940b9c8ad7d4ea7c660fa912fd6da 127.0.0.1:6379 myself,master - 0 0 1 connected 0-5
```

这里设置了三个节点，还有三个节点没有使用，作为一个完整的集群，每个负责处理槽的节点应该具有从节点，保证当它出现故障时可以自动进行故障转移。集群模式下，Redis节点角色分为主节点和从节点。首次启动的节点和被分配槽的节点都是主节点， 从节点负责复制主节点槽信息和相关的数据。 使用cluster replicate{nodeId}命令让一个节点成为从节点。
```
127.0.0.1:6382> cluster replicate 94756f9966f940b9c8ad7d4ea7c660fa912fd6da
127.0.0.1:6383> cluster replicate d67db857f8d6549b74c8e4b5b2cc8a78dfa918d3 
127.0.0.1:6384> cluster replicate 68bb0474810018d588582bd5a782ad0abcccdd30
```
Redis集群模式下的主从复制使用了之前介绍的Redis复制流程， 依然支持全量和部分复制。 复制（replication） 完成后， 整个集群的结构如图10-11所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200427100605792.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

## 用redis-trib.rb搭建集群







