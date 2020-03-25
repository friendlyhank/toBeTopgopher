## 前言:
**Redis支持RDB和AOF两种持久化机制**， 持久化功能有效地避免因进程退出造成的数据丢失问题， 当下次重启时利用之前持久化的文件即可实现数据恢复。

## RDB介绍
按指定时间间隔把数据生成快照保存到硬盘的过程,触发RDB持久化过程分为手动触发和自动触发。

### 自动触发
RDB的配置参数在配置文件redis.conf
```powershell
#时间策略
save 900 1
save 300 10
save 60 10000
```
这里说一下save的时间策略配置默认是三个，拿其中一个做说明,其他同理：

 - save 900 1 表示900s内至少有一个键更改，就会触发产生一次快照。

如果要关闭RDB快照生成可以直接在时间策略最尾部加上

```powershell
save ""
```

为什么要设置这么多条规则,因为考虑到每个时段的读写请求不一定是均衡的,为了平衡性能和数据安全,我们可以自由定制什么情况下触发备份。所以这里就是根据自身Redis写入情况来进行合理配置。


```powershell
#文件名称
dbfilename dump.rdb
#文件保存路径
dir ./

#压缩：默认采用LZF算法对生成的RDB文件做压缩处理,压缩会消耗CPU,但可大幅降低#文件体积
rdbcompression yes

#默认情况下，如果Redis在后台生成快照的时候失败，那么就会停止接收数据，目的#是让用户能知道数据没有持久化成功。但是如果你有其他的方式可以监控到Redis及#其持久化的状态，那么可以把这个功能禁止掉。
stop-writes-on-bgsave-error yes

#导入时是否校验
rdbchecksum yes
```

默认Redis会把快照文件存储为当前目录下一个名为**dump.rdb**的文件

### 手动触发save和bgsave
 - save命令：阻塞当前Redis服务器，直到RDB过程完成为止,对于内存比较大的实例会造成长时间阻塞,线上环境不建议使用。
 - bgsave命令：Redis进程执行fork操作创建子进程， RDB持久化过程由子进程负责， 完成后自动结束。 阻塞只发生在fork阶段， 一般时间很短。

显然bgsave命令是对save阻塞问题进行的优化,我们要重点看看bgsave的工作流程。

### bgsave工作流程
![bgsave工作流程](https://img-blog.csdnimg.cn/20200324174404884.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 1. 执行bgsave命令，Redis父进程判断当前是否存在正在执行的子进程,如RDB/AOF子进程，如果存在bgsave命令则直接返回。
 2. 父进程执行fork操作创建子进程，fork操作过程中父进程会阻塞，通过info stats命令查看latest_fork_usec选项，可以获取最近一个fork操作的耗时，单位为微秒。
 3. 父进程fork完成后， bgsave命令返回“Background saving started”信息并不再阻塞父进程， 可以继续响应其他命令。
 4. 子进程创建RDB文件， 根据父进程内存生成临时快照文件， 完成后对原有文件进行原子替换。 执行lastsave命令可以获取最后一次生成RDB的时间， 对应info统计的rdb_last_save_time选项。
 5. 进程发送信号给父进程表示完成， 父进程更新统计信息， 具体见info Persistence下的rdb_*相关选项。

除了执行命令手动触发之外，Redis内部还存在自动触发RDB的持久化 机制，例如以下场景： 
 - 使用save相关配置，如“save m n”。表示m秒内数据集存在n次修改 时，自动触发bgsave。
 - 如果从节点执行全量复制操作，主节点自动执行bgsave生成RDB文件并发送给从节点。 
 - 执行debug reload命令重新加载Redis时，也会自动触发save操作。 
 - 默认情况下执行shutdown命令时，如果没有开启AOF持久化功能则 自动执行bgsave。

### 优点
 - RDB是一个紧凑压缩的二进制文件,代表Redis在某个时间点上的数据快照。非常适用于备份，全量复制等场景。比如6小时执行bgsave备份。
 - 基于上面所描述的特性，RDB很适合用于灾备。单文件很方便就能传输到远程的服务器上。
 - RDB的性能很好，需要进行持久化时，主进程会fork一个子进程出来，然后把持久化的工作交给子进程，自己不会有相关的I/O操作。
 - Redis加载RDB恢复数据远远快于AOF的方式。

### 缺点
 - RDB容易造成数据的丢失。假设每5分钟保存一次快照，如果Redis因为某些原因不能正常工作，那么从上次产生快照到Redis出现问题这段时间的数据就会丢失了。
 - ·RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运 行都要执行fork操作创建子进程，属于重量级操作，频繁执行成本过高。
 - ·RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式 的RDB版本，存在老版本Redis服务无法兼容新版RDB格式的问题。


针对RDB不适合实时持久化的问题,Redis提供了AOF持久化方式来解决。

## AOF介绍
以独立日志的方式记录每次写命令.重启时再重新执行AOF文件中的命令达到恢复数据的目的。AOF的主要作用是解决了数据持久化的实时性， 目前已经是Redis持久化的主流方式。 
### AOF文件配置
详看配置文件redis.conf
```powershell
#是否开启AOF（yes or no）
appendonly yes 

#文件名称
appendfilename "appendonly.aof"

#文件保存路径,与RDB共用
dir ./
```
默认Redis会把文件存储为当前目录下一个名为**appendonly.aof**的文件

```powershell
#同步频率
# appendfsync always 
appendfsync everysec
# appendfsync no
```
特地把AOF的同步频率的配置拿出来讲,redis调用fsync的频率分三个：
 - appendfsync always  每次将新命令附加到AOF，非常慢,非常安全。（这里安全指的就是进程挂掉时候数据丢失的安全性)
 - 每秒fsync一次。速度快(再2.4版本中与快照方式的速度差不多),安全性不错(最多丢失1秒的数据)
 - 从不fsync,交给系统处理。速度非常快,但安全性一般,通常,Linux使用此配置30秒刷新一次数据。

默认采取的策略是fsync每秒执行一次,即快速又安全。

### 工作流程
AOF的工作流程操作： 命令写入（append）、文件同步（sync） 、 文件重写（rewrite）、重启加载
（load）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200325085432176.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
**流程如下：**
 1. 所有的写入命令会追加到aof_buf(缓冲区)中。
 2. AOF缓存区根据对应的策略向磁盘做同步操作。
 3. 随着AOF文件越来越大，需要定期对AOF文件进行重写，达到压缩的目的。
 4. 当Redis服务器重启时，可以加载AOF文件进行数据恢复。

### 优点
 - 比RDB可靠。你可以制定不同的fsync策略：不进行fsync、每秒fsync一次和每次查询进行fsync。默认是每秒fsync一次。这意味着你最多丢失一秒钟的数据。
 - AOF日志文件是一个纯追加的文件。就算是遇到突然停电的情况，也不会出现日志的定位或者损坏问题。甚至如果因为某些原因（例如磁盘满了）命令只写了一半到日志文件里，我们也可以用redis-check-aof这个工具很简单的进行修复。
 - 当AOF文件太大时，Redis会自动在后台进行重写。重写很安全，因为重写是在一个新的文件上进行，同时Redis会继续往旧的文件追加数据。新文件上会写入能重建当前数据集的最小操作命令的集合。当新文件重写完，Redis会把新旧文件进行切换，然后开始把数据写到新文件上。
 - AOF把操作命令以简单易懂的格式一条接一条的保存在文件里，很容易导出来用于恢复数据。例如我们不小心用FLUSHALL命令把所有数据刷掉了，只要文件没有被重写，我们可以把服务停掉，把最后那条命令删掉，然后重启服务，这样就能把被刷掉的数据恢复回来。

### 缺点
 - 在相同的数据集下，AOF文件的大小一般会比RDB文件大。
 - 在某些fsync策略下，AOF的速度会比RDB慢。通常fsync设置为每秒一次就能获得比较高的性能，而在禁止fsync的情况下速度可以达到RDB的水平。
 - 在过去曾经发现一些很罕见的BUG导致使用AOF重建的数据跟原数据不一致的问题。

### 重写机制
随着命令不断写入AOF， 文件会越来越大， 为了解决这个问题， Redis引入AOF重写机制压缩文件体积。
重写后的AOF文件会变小，原因如下：

 - 进程内已经超时的数据不再写入文件。
 - 旧的AOF文件含有无效命令， 如del key1、 hdel key2、 srem keys、 set a111、 set a222等。 重写使用进程内数据直接生成， 这样新的AOF文件只保留最终数据的写入命令。
 - 多条写命令可以合并为一个， 如： lpush list a、 lpush list b、 lpush list c可以转化为： lpush list a b c。 为了防止单条命令过大造成客户端缓冲区溢出， 对于list、 set、 hash、 zset等类型操作， 以64个元素为界拆分为多条。

AOF重写降低了文件占用空间， 除此之外， 另一个目的是： 更小的AOF文件可以更快地被Redis加载。

AOF重写过程可以手动触发和自动触发：
**自动触发**
```powershell
auto-aof-rewrite-min-size 64mb
```
表示运行AOF重写时文件最小体积， 默认为64MB。
```powershell
auto-aof-rewrite-percentage 100
```
代表当前AOF文件空间（ aof_current_size） 和上一次重写后AOF文件空间（ aof_base_size） 的比
值。

自动触发时机=aof_current_size>auto-aof-rewrite-minsize&&（ aof_current_size-aof_base_size） /aof_base_size>=auto-aof-rewritepercentage
表示触发重写的条件是文件大小最小为64mb,并且aof文件大小超过上一次重写文件的百分之百时会触发重写。

**手动触发**
手动触发直接调用bgrewriteaof命令。

#### bgrewriteaof工作流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200325112307515.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

 1. 执行AOF重写请求。如果当前进程正在执行AOF重写， 请求不执行并返回。如果当前进程正在执行bgsave操作， 重写命令延迟到bgsave完成之后再执行。
 2. 父进程执行fork创建子进程， 开销等同于bgsave过程。
 3. 主进程fork操作完成后， 继续响应其他命令。 所有修改命令依然写入AOF缓冲区并根据appendfsync策略同步到硬盘， 保证原有AOF机制正确性。
 4. 由于fork操作运用写时复制技术， 子进程只能共享fork操作时的内存数据。 由于父进程依然响应命令， Redis使用“AOF重写缓冲区”保存这部分新数据， 防止新AOF文件生成期间丢失这部分数据。
 5. 子进程根据内存快照， 按照命令合并规则写入到新的AOF文件。 每次批量写入硬盘数据量由配置aof-rewrite-incremental-fsync控制， 默认为32MB， 防止单次刷盘数据过多造成硬盘阻塞。
 6. 新AOF文件写入完成后， 子进程发送信号给父进程， 父进程更新统计信息， 具体见info persistence下的aof_*相关统计。
 7. 父进程把AOF重写缓冲区的数据写入到新的AOF文件。
 8. 使用新AOF文件替换老文件， 完成AOF重写。

## 重启加载
AOF和RDB文件都可以用于服务器重启时的数据恢复。
**加载流程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200325114851255.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
1. AOF持久化开启且存在AOF文件时， 优先加载AOF文件。
2. AOF关闭或者AOF文件不存在时， 加载RDB文件。
3. 加载AOF/RDB文件成功后， Redis启动成功。
4. AOF/RDB文件存在错误时， Redis启动失败并打印错误信息。

[官方参考](https://redis.io/topics/persistence)
Redis开发与运维书中写的非常详细

更多系统精彩的讲解：
[github](https://github.com/friendlyhank/toBeTopgopher)






