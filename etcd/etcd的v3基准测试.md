## 了解etcd性能
影响性能的两大因素:
 - **延迟**延迟是完成操作所花费的时间
 -**吞吐量**一段时间内完成的所有操作
当etcd接受并发的客户端请求时，平均延迟会随着总体吞吐量的增加而增加。在通常的云环境，比如 Google Compute Engine (GCE) 标准的 n-4 或者 AWS 上相当的机器类型，一个三成员 etcd 集群在轻负载下可以在低于 1 毫秒内完成一个请求，并在重负载下可以每秒完成超过 30000 个请求。
etcd使用raft一致性算法来在成员之间复制请求并达成一致。一致性性能，特别是提交延迟，受限于两个物理元素：**网络IO延迟和磁盘IO延迟。**完成etcd请求的最短时间是成员之间的网络往返时间(RTT)加上数据持久化存储fdatasync时间。数据中心内的RTT可能长达数百微妙。传统机械硬盘的典型fdatasync延迟是大概10ms。对于SSD，延迟通常低于1ms。为了提高吞吐量，etcd将多个请求一起批处理，然后将它们提交给Raft。通过此批处理策略，尽管负载很大,etcd仍可实现高吞吐量。
除此之外，其他子系统也会影响etcd的整体性能，每个序列化的etcd请求都必须通过etcd的boltdb支持的MVCC存储引擎运行，这通常需要数十微妙才能完成。etcd定期写入增量快照，并将其与先前磁盘快照合并，此过程也可能会导致延迟峰值。同样，进行中的压缩也会影响etcd的性能，幸运得是，由于压缩是错开的，因此影响通常是微不足道的，因此它不会与常规请求竞争资源。RPC系统grpc为etcd定义了明确、可扩展的API,但它也会引入额外的延迟，尤其是对于本地的读取。

## 基准测试
etcd自带基准测试在目录tool/benchmark,可以通过go install安装后在go环境bin目录下。
```
# write to leader
benchmark --endpoints=${HOST_1} --target-leader --conns=1 --clients=1 \
    put --key-size=8 --sequential-keys --total=10000 --val-size=256
benchmark --endpoints=${HOST_1} --target-leader  --conns=100 --clients=1000 \
    put --key-size=8 --sequential-keys --total=100000 --val-size=256

# write to all members
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 \
    put --key-size=8 --sequential-keys --total=100000 --val-size=256

benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 \
    range YOUR_KEY --consistency=s --total=100000
```
 - target-leader目标服务器：仅领袖
 - conns 连接数
 - clients客户端数量
 - endpoints指定服务端
 - key-size键的大小(以字节为单位)
 - val-size值大小(以字节为单位)
 - total 键的总数
 - consistency 一致性(l表示线性化,s表示串行化)

输出结果：
```
 10000 / 10000 Booooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo! 100.00% 34s

Summary:
  Total:        34.5578 secs.
  Slowest:      0.3055 secs.
  Fastest:      0.0000 secs.
  Average:      0.0034 secs.
  Stddev:       0.0036 secs.
  Requests/sec: 289.3703

Response time histogram:
  0.0000 [245]  |∎
  0.0305 [9753] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  0.0611 [1]    |
  0.0916 [0]    |
  0.1222 [0]    |
  0.1527 [0]    |
  0.1833 [0]    |
  0.2138 [0]    |
  0.2444 [0]    |
  0.2749 [0]    |
  0.3055 [1]    |

Latency distribution:
  10% in 0.0029 secs.
  25% in 0.0030 secs.
  50% in 0.0030 secs.
  75% in 0.0039 secs.
  90% in 0.0040 secs.
  95% in 0.0040 secs.
  99% in 0.0140 secs.
  99.9% in 0.0190 secs.
```

[官方原文](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/performance.md#benchmarks)

