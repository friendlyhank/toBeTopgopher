(1)sentinel masters
```
sentinel masters
```
显示所有被监控的主节点状态以及统计信息

(2)sentinel master  (master name)
```
sentinel master mymaster
```
展示指定(master name)的主节点状态以及相关的统计信息。

(3)sentinel slaves (master name)
```
sentinel slaves mymaster
```
展示指定(master name)的从节点状态以及相关的统计信息。

(4)sentinel sentinels (master name)
```
sentinel sentinels mymaster
```
展示指定(master name)的Sentinel节点集合(不包含当前Sentinel节点)

(5)sentinel get-master-addr-by-name (master name)
```
sentinel get-master-addr-by-name mymaster
```
返回指定(master name)主节点的IP地址和端口。

(6)sentinel reset (pattern)
```
sentinel reset mymaster
```
当前sentinel节点对符合(pattern)(通配符风格)主节点的配置进行重置，包含清除主节点的相关状态(例如故障转移),重新发现从节点和Sentinel节点。

(7)sentinel failover (master name)
```
sentinel failover mymaster
```
对指定mymaster主节点进行强制故障转移(没有和其他Sentinel节点"协商"),当故障转移完成后，其他Sentinel节点按照故障转移的结果更新自身配置。

sentinel会选定一个从节点替换。

(8)sentinel ckquorum (master name)
```
sentinel ckquorum mymaster
```
检测当前可达的Sentinel节点总数是否达到(quorum)的个数。例如quorum=3,而当前可达的Sentinel节点个数为2个， 那么将无法进行故障转移，Redis Sentinel的高可用特性也将失去。

(9)sentinel flushconfig
```
sentinel flushconfig
```
将Sentinel节点的配置强制刷到磁盘上，这个命令Sentinel节点自身用得比较多，对于开发和运维人员只有当外部原因(磁盘损坏)造成配置文件损坏或者丢失时，这个命令时有用的。

(10)sentinel remove (master name)
取消当前Sentinel节点对于指定(master name)主节点的监控
```
sentinel remove mymaster
```

(11)sentinel monitor<master name><ip><port><quorum>
```
sentinel monitor mymaster 192.168.85.109 6379 2
```

(12)sentinel set (master name)

(13)sentinel is-master-down-by-addr
Sentinel节点之间用来交换对主节点是否下线的判断， 根据参数的不同， 还可以作为Sentinel领导者选举的通信方式。


