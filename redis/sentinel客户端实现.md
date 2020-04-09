
客户端的实现要从多个Sentinel节点入手，如果只对主Sentinel，那么主节点要是挂掉了，客户端就无法获取这个变化。

要正确的连接Redis Sentinel,必须有Sentinel节点集合和masterName两个参数。

## 客户端的基本实现原理
 1. 实现一个Redis Sentinel客户端的基本步骤如下：
遍历Sentinel节点集合获取一个可用的Sentinel节点， 后面会介绍Sentinel节点之间可以共享数据， 所以
 从任意一个Sentinel节点获取主节点信息都是可以的
 2. 通过sentinel get-master-addr-by-name master-name这个API来获取对应主节点的相关信息
 3. 验证当前获取的“主节点”是真正的主节点， 这样做的目的是为了防止故障转移期间主节点的变化
 4. 保持和Sentinel节点集合的“联系”， 时刻获取关于主节点的相关“信息”

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409090620181.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409090652120.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409090746457.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200409090900171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

从上面的模型可以看出， Redis Sentinel客户端只有在初始化和切换主节点时需要和Sentinel节点集合进行交互来获取主节点信息， **所以在设计客户端时需要将Sentinel节点集合考虑成配置（ 相关节点信息和变化） 发现服务。**

**具体过程如下：**
 1. 遍历Sentinel节点集合(可以服务发现方式实现)， 找到一个可用的Sentinel节点， 如果找不到就从Sentinel节点集合中去找下一个， 如果都找不到直接抛出异常给客户端
 2. 找到一个可用的Sentinel节点， 执行sentinelGetMasterAddrByName（masterName） ， 找到对应主节点信息。
 3. 获取节点角色信息
 4. 为每一个Sentinel节点单独启动一个线程， 利用Redis的发布订阅功能， 每个线程订阅Sentinel节点上切换master的相关频道+switch-master。

