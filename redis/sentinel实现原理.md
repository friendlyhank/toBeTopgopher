## 三个定时监控任务
redis Sentinel通过三个定时监控任务完成对各个节点发现和监控：

 - 每隔10秒， 每个Sentinel节点会向主节点和从节点发送info命令获取最新的拓扑结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409130051898.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
这个定时任务的作用具体可以表现在三个方面：
·通过向主节点执行info命令， 获取从节点的信息， 这也是为什么Sentinel节点不需要显式配置监控从节点。
·当有新的从节点加入时都可以立刻感知出来。
·节点不可达或者故障转移后， 可以通过info命令实时更新节点拓扑信息

(2)每隔2秒， 每个Sentinel节点会向Redis数据节点的__sentinel__： hello 频道上发送该Sentinel节点对于主节点的判断以及当前Sentinel节点的信息,同时每个Sentinel节点也会订阅该频道， 来了解其他
Sentinel节点以及它们对主节点的判断， 所以这个定时任务可以完成以下两个工作：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020040913033196.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 - 发现新的Sentinel节点： 通过订阅主节点的__sentinel__： hello了解其他的Sentinel节点信息， 如果是新加入的Sentinel节点， 将该Sentinel节点信息保存起来， 并与该Sentinel节点创建连接。
 - Sentinel节点之间交换主节点的状态， 作为后面客观下线以及领导者选举的依据。

Sentinel节点publish的消息格式如下：
<Sentinel节点IP> <Sentinel节点端口> <Sentinel节点runId> <Sentinel节点配置版本>
<主节点名字> <主节点Ip> <主节点端口> <主节点配置版本>

(3)每隔1秒， 每个Sentinel节点会向主节点、 从节点、 其余Sentinel节点发送一条ping命令做一次心跳检测， 来确认这些节点当前是否可达。Sentinel节点对主节点、 从节点、 其余Sentinel节点都建立起连接， 实现了对每个节点的监控， 这个定时任务是节点失败判定的重要依据。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409161941360.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

## 主观下线和客观下线
1.主观下线
第三个定时任务， 每个Sentinel节点会每隔1秒对主节点、 从节点、 其他Sentinel节点发送ping命令做心跳检测， 当这些节点超过down-after-milliseconds没有进行有效回复， Sentinel节点就会对该节点做失败
判定， 这个行为叫做主观下线。 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409163442228.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

2.客观下线
当Sentinel主观下线的节点是主节点时， 该Sentinel节点会通过sentinel ismaster-down-by-addr命令向其他Sentinel节点询问对主节点的判断， 当超过（quorum)个数， Sentinel节点认为主节点确实有问题， 这时该Sentinel节点会做出客观下线的决定， 这样客观下线的含义是比较明显了， 也就是大部分
Sentinel节点都对主节点的下线做了同意的判定， 那么这个判定就是客观的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409163546776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

这里有必要对sentinel is-master-down-by-addr命令做一个介绍
```
sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```
 - ip： 主节点IP。
 - port： 主节点端口。
 - current_epoch： 当前配置纪元。
 - runid： 此参数有两种类型， 不同类型决定了此API作用的不同。
	当runid等于“*”时， 作用是Sentinel节点直接交换对主节点下线的判定。
	当runid等于当前Sentinel节点的runid时， 作用是当前Sentinel节点希望目标Sentinel节点同意自己成为领导者的请求， 有关Sentinel领导者选举。

例如sentinel-1节点对主节点做主观下线后， 会向其余Sentinel节点（假设sentinel-2和sentinel-3节点） 发送该命令：
```
sentinel is-master-down-by-addr 127.0.0.1 6379 0 *
```
返回结果包含三个参数， 如下所示：
down_state： 目标Sentinel节点对于主节点的下线判断， 1是下线， 0是在线。
·leader_runid： 当leader_runid等于“*”时， 代表返回结果是用来做主节点是否不可达， 当leader_runid等于具体的runid， 代表目标节点同意runid成为领导者。
·leader_epoch： 领导者纪元。

##  领导者Sentinel节点选举
假如Sentinel节点对于主节点已经做了客观下线， 那么是不是就可以立即进行故障转移了？ 当然不是， 实际上故障转移的工作只需要一个Sentinel节点来完成即可， 所以Sentinel节点之间会做一个领导者选举的工作， 选出一个Sentinel节点作为领导者进行故障转移的工作。 Redis使用了Raft算法实现领导者选举， 因为Raft算法相对比较抽象和复杂， 以及篇幅所限， 所以这里给出一个Redis Sentinel进行领导者选举的大致思路：
 1. 每个在线的Sentinel节点都有资格成为领导者， 当它确认主节点主观下线时候， 会向其他Sentinel节点发送sentinel is-master-down-by-addr命令，要求将自己设置为领导者。
 2. 收到命令的Sentinel节点， 如果没有同意过其他Sentinel节点的sentinel
is-master-down-by-addr命令， 将同意该请求， 否则拒绝。
 3. 如果该Sentinel节点发现自己的票数已经大于等于max（quorum，
num（sentinels） /2+1） ， 那么它将成为领导者。
 4. 如果此过程没有选举出领导者， 将进入下一次选举。

## 故障转移
领导者选举出的Sentinel节点负责故障转移， 具体步骤如下：
 1. 在从节点列表中选出一个节点作为新的主节点， 选择方法如下：
 a)过滤： “不健康”（主观下线、 断线） 、 5秒内没有回复过Sentinel节
点ping响应、 与主节点失联超过down-after-milliseconds*10秒。
b)选择slave-priority（从节点优先级） 最高的从节点列表， 如果存在则返回， 不存在则继续。
c)选择复制偏移量最大的从节点（复制的最完整） ， 如果存在则返回， 不存在则继续。
d)选择runid最小的从节点。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409220406581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 2. Sentinel领导者节点会对第一步选出来的从节点执行slaveof no one命
令让其成为主节点。
 3. Sentinel领导者节点会向剩余的从节点发送命令， 让它们成为新主节点的从节点， 复制规则和parallel-syncs参数有关。
 4. Sentinel节点集合会将原来的主节点更新为从节点， 并保持着对其关注， 当其恢复后命令它去复制新的主节点。

