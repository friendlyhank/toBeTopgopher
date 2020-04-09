## 基本概念
redis Sentinel为了解决Redis高可用的问题，当主节点出现故障时，Redis Sentinel能自动完成故障发现和故障的转移。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406180844609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

故障转移处理的步骤：
 
 1. 主节点出现故障，此时两个从节点与主节点失去连接，主从复制失败。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406181451781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 2. 每个Sentinel节点通过定期监控发现主节点出现了故障。
 3. 多个Sentinel节点对主节点的故障达成一致，选举出seltinel-3节点作为领导负责故障转移。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406181936225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406182001791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 4. Sentinel领导者节点执行了故障转移，整个过程是自动化完成的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406182316903.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 5. 故障转移后整个Redis Sentinel的拓扑结构如图所示：
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200406182520152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
**redis Sentinel的功能**：
**监控**：Sentinel节点会定期检测Redis数据节点、其余Sentinel节点是否可达。
**通知**：Sentinel节点会将故障转移的结果通知给应用方。
**主节点故障转移**：实现从节点晋升为主节点并维护后续正确的主从关系。
**配置提供者**：在Redis Sentinel结构中，客户端初始化的时候连接的是Sentinel节点集合，从中获取主节点信息。

多个Sentinel节点，这样做带来两个好处：对于节点的故障判断是由多个Sentinel节点共同完成的，这样可以有效防止误判。
Sentinel节点集合是由若干个Sentinel节点组成的，这样即使个别Sentinel节点不可用，整个Sentinel节点集合依然健壮。

## 哨兵的搭建和安装
这里省略主从复制的搭建
| IP |Port  | 角色 |
|--|--|--|
| 192.168.85.109 |6379 |redis master|
| 192.168.85.110 |6379 |redis slave|
| 192.168.85.111 |6379 |redis slave|
| 192.168.85.109 |26379 |Sentinel|
| 192.168.85.110 |26379 |Sentinel|
| 192.168.85.111 |26379  |Sentinel|

修改端口、logfile、
```go
port 26379
daemonize yes
logfile "26379.log"
dir /opt/soft/redis/data
sentinel monitor mymaster 192.168.85.109 6379 2
sentinel auth-pass mymaster 123456
```
注意如果redis有设置密码,sentinel配置也要设置密码。

Sentinel启动成功之后，配置也会发生改变：
在配置最底部：
```
//发现了两个slave节点
sentinel known-replica mymaster 192.168.85.111 6379
sentinel known-replica mymaster 192.168.85.110 6379
//发现了两个sentinel节点
sentinel known-sentinel mymaster 192.168.85.111 26379 337b1ebd37cd6e3a63c49fd4da4a1102fc8fd368
sentinel known-sentinel mymaster 192.168.85.110 26379 0cd67c4a7793e1bd098d308c38a956244ecc09f5
```

(1)sentinel monitor
```
sentinel monitor <master-name> <ip> <port> <quorum>
```
Sentinel节点会定期监控主节点，所以从配置上必然也会有所体现，本配置说明Sentinel节点要监控的是一个名字叫<master-name>,ip地址和端口为<ip><port>的主节点。<quorum>代表要判定主节点最终不可达所需要的票数。实际上Sentinel节点会对所有节点进行监控，在Sentinel节点配置中没有看到从节点和其余Sentinel节点的相关信息。

quorum用于**故障发现和判断**,表示至少要多少个Sentinel节点认为主节点不可达，才进行故障转移
quorum还用于**领导者的选举**,至少要有max（quorum， num（sentinels） /2+1） 个Sentinel节点参与选举， 才能选出领导者Sentinel,从而完成故障转移。例如有5个Sentinel节点,quorum=4,那么至少需要
max(4,5/2+1)=4

(2)sentinel down-after-milliseconds
```
sentinel down-after-milliseconds <master-name> <times>
```
每个Sentinel节点都要通过定期发送ping命令来判断Redis数据节点和其余Sentinel节点是否可达，如果超过sentinel down-after-milliseconds配置的时间且没有有效回复，则判定节点不可达。这个配置是对节点失败判定的重要依据。

**如果sentinel down-after-milliseconds越大则说明越宽松，反之则越严格。条件宽松带来的问题是等待故障转移的时间会很长，反之，条件严格则有可能导致误判。**

down-after-milliseconds虽然以<master-name>为参数， 但实际上对Sentinel节点、 主节点、 从节点的失败判定同时有效。

(3)sentinel parallel-syncs
```
sentinel parallel-syncs <master-name> <nums>
```
当Sentinel节点集合对主节点故障判定达成一致时， Sentinel领导者节点会做故障转移操作， 选出新的主节点， 原来的从节点会向新的主节点发起复制操作， parallel-syncs就是用来限制在一次故障转移之后， 每次向新的主节点发起复制操作的从节点个数。 如果这个参数配置的比较大， 那么多个从节点会向新的主节点同时发起复制操作， 尽管复制操作通常不会阻塞主节点，但是同时向主节点发起复制， 必然会对主节点所在的机器造成一定的网络和磁盘IO开销。 

(4)sentinel failover-timeout
```
sentinel failover-timeout <master-name> <times>
```
failover-timeout通常被解释成故障转移超时时间， 但实际上它作用于故障转移的各个阶段：
a） 选出合适从节点。
b） 晋升选出的从节点为主节点。
c） 命令其余从节点复制新的主节点。
d） 等待原主节点恢复后命令它去复制新的主节点。

failover-timeout的作用具体体现在四个方面：
 1. 如果Redis Sentinel对一个主节点故障转移失败， 那么下次再对该主节点做故障转移的起始时间是failover-timeout的2倍。
 2. 在b） 阶段时， 如果Sentinel节点向a） 阶段选出来的从节点执行
499slaveof no one一直失败（例如该从节点此时出现故障） ， 当此过程超过
failover-timeout时， 则故障转移失败。
 3. 在b） 阶段如果执行成功， Sentinel节点还会执行info命令来确认a）阶段选出来的节点确实晋升为主节点， 如果此过程执行时间超过failovertimeout时， 则故障转移失败。
 4. 如果c） 阶段执行时间超过了failover-timeout（不包含复制时间） ，则故障转移失败。 注意即使超过了这个时间， Sentinel节点也会最终配置从节点去同步最新的主节点。
 
(5) sentinel auth-pass 密码设置


**sentinel可以设置监控多个主节点**，只需要指定多个masterName来区分主节点即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200408213641996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
Sentinel可以监控一个主节点，也可以监控多个主节点。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200408214918945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200408214934101.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
那么在实际生产环境中更偏向于哪一种部署方式呢， 下面分别分析两种方案的优缺点。
方案一： 一套Sentinel， 很明显这种方案在一定程度上降低了维护成本， 因为只需要维护固定个数的Sentinel节点， 集中对多个Redis数据节点进行管理就可以了。 但是这同时也是它的缺点， 如果这套Sentinel节点集合出现异常， 可能会对多个Redis数据节点造成影响。 还有如果监控的Redis数据
节点较多， 会造成Sentinel节点产生过多的网络连接， 也会有一定的影响。

方案二： 多套Sentinel， 显然这种方案的优点和缺点和上面是相反的，每个Redis主节点都有自己的Sentinel节点集合， 会造成资源浪费。 但是优点也很明显， 每套Redis Sentinel都是彼此隔离的。
