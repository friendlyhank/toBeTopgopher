## 节点标志
```powershell
–name
```
 - 此成员的可读名称
 - 默认值："default"
 - 环境变量：ETCD_NAME

```powershell
–data-dir
```
 - 数据目录的路径
 - 环境变量：ETCD_WAL_DIR

```powershell
–wal-dir
```
 - 专用wal目录的路径。如果设置了此标志,etcd会将WAL文件写入walDir而不是dataDir。这允许使用专用磁盘，并有助于避免日志记录和其他IO操作之间的io竞争。

```powershell
–snapshot-count
```
 - 触发快照到磁盘的已提交事务数
 - 默认值："100000"
 - 环境变量:ETCD_SNAPSHOT_COUNT

```powershell
–heartbeat-interval
```
 - 心跳间隔的时间(以毫秒为单位)
 - 默认值:"100"
 - 环境变量:ETCD_HEARTBEAT_INTERVAL

```powershell
–election-timeout
```
 - 选举超时的时间(以毫秒为单位)
 - 默认值："1000"
 - 环境变量：ETCD_ELECTION_TIMEOUT

```powershell
–listen-peer-urls
```
 - 监听集群的URL列表
 - 默认值："http：// localhost：2380"
 - 环境变量：ETCD_LISTEN_PEER_URLS
 - 例如："http://10.0.0.1:2380"
 - 无效的示例："http://example.com：2380"(域名对于绑定无效)

```powershell
–listen-client-urls
```
 - 监听客户端的URL列表
 - 默认值：" http：// localhost：2379"
 - 环境变量：ETCD_LISTEN_CLIENT_URLS
 - 例如："" http://10.0.0.1:2379"
 - 无效的示例：" http://example.com：2379"(域名对于绑定无效)

```powershell
–max-snapshots
```
 - 保留快照文件的最大数量(0为无限)
 - 默认值：5
 - 环境变量：ETCD_MAX_SNAPSHOTS
 
```powershell
 –max-wals
```
 - 保留的wal文件的最大数量(0为无限)
 - 默认值：5
 - 环境变量：ETCD_MAX_WALS

```powershell
–cors
```
 - 逗号分隔的CORS来源白名单（跨来源资源共享）。
- 默认值：“”
- 环境变量：ETCD_CORS

```powershell
–quota-backend-bytes
```
- 后端大小超过给定配额时引发警报（0默认为低空间配额）。
- 默认值：0
- 环境变量：ETCD_QUOTA_BACKEND_BYTES

```powershell
–backend-batch-limit
```
- BackendBatchLimit是提交后端事务之前的最大操作。
- 默认值：0
- 环境变量：ETCD_BACKEND_BATCH_LIMIT

```powershell
–backend-bbolt-freelist-type
```
- etcd后端（bboltdb）使用的自由列表类型（支持数组和映射的类型）。
- 默认值：map
- 环境变量：ETCD_BACKEND_BBOLT_FREELIST_TYPE

```powershell
–backend-batch-interval
```
- BackendBatchInterval是提交后端事务之前的最长时间。
- 默认值：0
- 环境变量：ETCD_BACKEND_BATCH_INTERVAL

```powershell
–max-txn-ops
```
- 交易中允许的最大操作数。
- 默认值：128
- 环境变量：ETCD_MAX_TXN_OPS

```powershell
–max-request-bytes
```
- 服务器将接受的最大客户端请求大小（以字节为单位）。
- 默认值：1572864
- 环境变量：ETCD_MAX_REQUEST_BYTES

```powershell
–grpc-keepalive-min-time
```
- 客户端在ping服务器之前应等待的最小持续时间间隔。
- 默认值：5秒
- 环境变量：ETCD_GRPC_KEEPALIVE_MIN_TIME

```powershell
–grpc-keepalive-interval
```
- 服务器到客户端ping的频率持续时间，以检查连接是否有效（0禁用）。
- 默认值：2h
- 环境变量：ETCD_GRPC_KEEPALIVE_INTERVAL

```powershell
–grpc-keepalive-timeout
```
- 关闭无响应的连接之前的额外等待时间（0禁用）。
- 默认值：20秒
- 环境变量：ETCD_GRPC_KEEPALIVE_TIMEOUT

## 集群标志
--initial-advertise-peer-urls，--initial-cluster，--initial-cluster-state，和--initial-cluster-token标志在自举（使用静态自举， 发现服务引导或运行时重新配置）的一个新成员，和重新启动的现有成员时忽略。

--discovery使用发现服务时，需要设置前缀标志 。

```powershell
–initial-advertise-peer-urls
```
- 此成员的对等URL的列表，以通告到集群的其余部分。这些地址用于在集群周围传递etcd数据。所有集群成员必须至少有一个路由。这些URL可以包含域名。
- 默认值：“ http：// localhost：2380”
- 环境变量：ETCD_INITIAL_ADVERTISE_PEER_URLS
例如：“ http://example.com：2380，http://10.0.0.1:2380 ”

```powershell
–initial-cluster
```
- 用于引导的初始群集配置。
- 默认值：“默认值= http：// localhost：2380”
- 环境变量：ETCD_INITIAL_CLUSTER
密钥是所--name提供的每个节点的标志值。默认值default用于密钥，因为这是--name标志的默认值。

```powershell
–initial-cluster-state
```
- 初始群集状态（“new”或“existing”）。new对于初始静态或DNS引导过程中存在的所有成员，设置为。如果将此选项设置为existing，则etcd将尝试加入现有集群。如果设置了错误的值，则etcd将尝试启动但安全失败。
- 默认值：“new”
- 环境变量：ETCD_INITIAL_CLUSTER_STATE

```powershell
–initial-cluster-token
```
- 引导期间etcd群集的初始群集令牌。
- 默认值：“ etcd-cluster”
- 环境变量：ETCD_INITIAL_CLUSTER_TOKEN

```powershell
–advertise-client-urls
```
- 此成员的客户端URL的列表，用于播发给集群的其余部分。这些URL可以包含域名。
- 默认值：“ http：// localhost：2379”
- 环境变量：ETCD_ADVERTISE_CLIENT_URLS
- 例如：“ http://example.com：2379，http://10.0.0.1:2379 ”
- 请注意，如果从群集成员中发布诸如http：// localhost：2379之类的URL，并且正在使用etcd的代理功能。这将导致循环，因为代理将向其自身转发请求，直到其资源（内存，文件描述符）最终耗尽为止。

```powershell
–discovery
```
- 用于引导群集的发现URL。
- 默认值：“”
- 环境变量：ETCD_DISCOVERY

```powershell
–discovery-srv
```
- 用于引导群集的DNS srv域。
- 默认值：“”
- 环境变量：ETCD_DISCOVERY_SRV

```powershell
–discovery-srv-name
```
- 使用DNS引导时查询的DNS srv名称的后缀。
- 默认值：“”
- 环境变量：ETCD_DISCOVERY_SRV_NAME

```powershell
–discovery-fallback
```
- 发现服务失败时的预期行为（“退出”或“代理”）。“代理”仅支持v2 API。
- 默认值：“proxy”
- 环境变量：ETCD_DISCOVERY_FALLBACK

```powershell
–discovery-proxy
```
- HTTP代理，用于发现服务的流量。
- 默认值：“”
- 环境变量：ETCD_DISCOVERY_PROXY

```powershell
–strict-reconfig-check
```
- 拒绝可能导致仲裁丢失的重新配置请求。
- 默认值：true
- 环境变量：ETCD_STRICT_RECONFIG_CHECK

```powershell
–auto-compaction-retention
```
- mvcc密钥值存储的自动压缩保留时间（小时）。0表示禁用自动压缩。
- 默认值：0
- 环境变量：ETCD_AUTO_COMPACTION_RETENTION

```powershell
–auto-compaction-mode
```
- 解释“自动压缩保留”之一：“定期”，“修订”。基于周期的保留的“定期”，如果未提供时间单位（例如“ 5m”），则默认为小时。“修订”用于基于修订号的保留。
- 默认值：periodic
环境变量：ETCD_AUTO_COMPACTION_MODE

```powershell
–enable-v2
```
- 接受etcd V2客户请求
- 默认值：false
- 环境变量：ETCD_ENABLE_V2


## 代理标志
--proxy前缀标志将etcd配置为以代理模式运行 。“代理”仅支持v2 API。

```powershell
–proxy
```
- 代理模式设置（“关闭”，“只读”或“打开”）。
- 默认值：“off”
- 环境变量：ETCD_PROXY

```powershell
–proxy-failure-wait
```
- 在重新考虑端点请求之前，端点将保持故障状态的时间（以毫秒为单位）。
- 默认值：5000
- 环境变量：ETCD_PROXY_FAILURE_WAIT

```powershell
–proxy-refresh-interval
```
- 端点刷新间隔的时间（以毫秒为单位）。
- 默认值：30000
- 环境变量：ETCD_PROXY_REFRESH_INTERVAL

```powershell
–proxy-dial-timeout
```
- 拨号超时的时间（以毫秒为单位），或0以禁用超时
- 默认值：1000
- 环境变量：ETCD_PROXY_DIAL_TIMEOUT

```powershell
–proxy-write-timeout
```
- 写入超时的时间（以毫秒为单位），或禁用超时的时间为0。
- 默认值：5000
- 环境变量：ETCD_PROXY_WRITE_TIMEOUT

```powershell
–proxy-read-timeout
```
- 读取超时的时间（以毫秒为单位），或者为0以禁用超时。
- 如果使用手表，请勿更改此值，因为会使用较长的轮询请求。
- 默认值：0
- 环境变量：ETCD_PROXY_READ_TIMEOUT


## 安全标志
安全标志有助于构建安全的etcd集群。

```powershell
–cert-file
```
- 客户端服务器TLS证书文件的路径。
- 默认值：“”
- 环境变量：ETCD_CERT_FILE

```powershell
–key-file
```
- 客户端服务器TLS密钥文件的路径。
- 默认值：“”
- 环境变量：ETCD_KEY_FILE

```powershell
–client-cert-auth
```
- 启用客户端证书认证。
- 默认值：false
- 环境变量：ETCD_CLIENT_CERT_AUTH
- gRPC网关不支持CN身份验证。

```powershell
–client-crl-file
```
- 客户端证书吊销列表文件的路径。
- 默认值：“”
- 环境变量：ETCD_CLIENT_CRL_FILE

```powershell
–client-cert-allowed-hostname
```
- 允许客户端证书身份验证的允许TLS名称。
- 默认值：“”
- 环境变量：ETCD_CLIENT_CERT_ALLOWED_HOSTNAME

```powershell
–trusted-ca-file
```
- 客户端服务器TLS可信CA证书文件的路径。
- 默认值：“”
- 环境变量：ETCD_TRUSTED_CA_FILE

```powershell
–auto-tls
```
使用生成的证书的客户端TLS
默认值：false
环境变量：ETCD_AUTO_TLS

```powershell
–peer-cert-file
```
- 对等服务器TLS证书文件的路径。这是对等流量的证书，用于服务器和客户端。
- 默认值：“”
- 环境变量：ETCD_PEER_CERT_FILE

```powershell
–peer-key-file
```
- 对等服务器TLS密钥文件的路径。这是用于服务器和客户端的对等流量的关键。
- 默认值：“”
- 环境变量：ETCD_PEER_KEY_FILE

```powershell
–peer-client-cert-auth
```
- 启用对等客户端证书认证。
- 默认值：false
- 环境变量：ETCD_PEER_CLIENT_CERT_AUTH

```powershell
–peer-crl-file
```
- 对等证书吊销列表文件的路径。
- 默认值：“”
- 环境变量：ETCD_PEER_CRL_FILE

```powershell
–peer-trusted-ca-file
```
- 对等服务器TLS可信CA文件的路径。
- 默认值：“”
- 环境变量：ETCD_PEER_TRUSTED_CA_FILE

```powershell
–peer-auto-tls
```
- 使用生成的证书对等TLS
- 默认值：false
- 环境变量：ETCD_PEER_AUTO_TLS

```powershell
–peer-cert-allowed-cn
```
- 允许使用CommonName进行对等身份验证。
- 默认值：“”
- 环境变量：ETCD_PEER_CERT_ALLOWED_CN

```powershell
–peer-cert-allowed-hostname
```
- 允许的TLS证书名称用于内部对等身份验证。
- 默认值：“”
- 环境变量：ETCD_PEER_CERT_ALLOWED_HOSTNAME

```powershell
–cipher-suites
```
- 服务器/客户端和对等方之间受支持的TLS密码套件的逗号分隔列表。
- 默认值：“”
- 环境变量：ETCD_CIPHER_SUITES

## 记录标志
```powershell
–logger
```
从v3.4起可用。 警告：--logger=capnslog在v3.5中已弃用。
- 指定“ zap”用于结构化日志记录或“ capnslog”。
- 默认值：capnslog
- 环境变量：ETCD_LOGGER

```powershell
–log-outputs
```
- 指定“ stdout”或“ stderr”以跳过日志记录，即使在systemd或逗号分隔的输出目标列表下运行时也是如此。
- 默认值：默认
- 环境变量：ETCD_LOG_OUTPUTS
在zap logger偏头痛过程中，针对v3.4，“默认”使用“ stderr”配置

```powershell
–log-level
```
从v3.4起可用。
- 配置日志级别。仅支持调试，信息，警告，错误，紧急或致命。
- 默认值：信息
- 环境变量：ETCD_LOG_LEVEL
“默认”使用“信息”。

```powershell
–debug
```
警告：在v3.5中已弃用。

- 将所有子软件包的默认日志级别降低到DEBUG。
- 默认值：false（所有软件包的INFO）
- 环境变量：ETCD_DEBUG

```powershell
–log-package-levels
```
警告：在v3.5中已弃用。

- 将各个etcd子软件包设置为特定的日志级别。一个例子是etcdserver=WARNING,security=DEBUG
- 默认值：“”（所有软件包的INFO）
- 环境变量：ETCD_LOG_PACKAGE_LEVELS

## 不安全标志
使用不安全标志时请小心，因为它会破坏共识协议提供的保证。例如，如果群集中的其他成员仍然存在，可能会感到恐慌。使用这些标志时，请遵循说明。

```powershell
–force-new-cluster
```
- 强制创建一个新的单成员群集。它提交配置更改，以强制删除集群中的所有现有成员并添加自身，但是强烈建议不要这样做。请查看 灾难恢复文档以了解首选的v3恢复过程。
- 默认值：false
- 环境变量：ETCD_FORCE_NEW_CLUSTER

## 杂项标志

```powershell
–version
```
- 打印版本并退出。
- 默认值：false

```powershell
–config-file
```
- 从文件加载服务器配置。请注意，如果提供了配置文件，则其他命令行标志和环境变量将被忽略。
- 默认值：“”
- 示例： 样本配置文件
- env变量：ETCD_CONFIG_FILE

## 分析标志
```powershell
–enable-pprof
```
- 通过HTTP服务器启用运行时分析数据。地址位于客户端URL +“ / debug / pprof /”
- 默认值：false
- 环境变量：ETCD_ENABLE_PPROF

```powershell
–metrics
```
- 设置导出指标的详细程度，指定“广泛”以包括服务器端grpc直方图指标。
- 默认值：基本
- 环境变量：ETCD_METRICS

```powershell
–listen-metrics-urls
```
- 要侦听的其他URL列表，将同时响应/metrics和/health端点
- 默认值：“”
- 环境变量：ETCD_LISTEN_METRICS_URLS

## 验证标志
```powershell
–auth-token
```
- 指定令牌类型和特定于令牌的选项，尤其是对于JWT。其格式为“ type，var1 = val1，var2 = val2，…”。可能的类型为“ simple”或“ jwt”。可能的变量为“ sign-method”，用于指定jwt的符号方法（其可能值为“ ES256” '，'ES384'，'ES512'，'HS256'，'HS384'，'HS512'，'RS256'，'RS384'，'RS512'，'PS256'，'PS384'或'PS512'）， -key”用于指定用于验证jwt的公共密钥的路径；“ priv-key”用于指定用于对jwt进行签名的私有密钥的路径；“ ttl”用于指定jwt令牌的TTL。
对于非对称算法（“ RS”，“ PS”，“ ES”），公钥是可选的，因为私钥包含足够的信息来签名和验证令牌。
JWT的示例选项：'–auth-token令牌jwt，pub-key = app.rsa.pub，priv-key = app.rsa，sign-method = RS512，ttl = 10m'
- 默认值：“简单”
- 环境变量：ETCD_AUTH_TOKEN

```powershell
–bcrypt-cost
```
- 指定用于哈希认证密码的bcrypt算法的成本/强度。有效值在4到31之间。
- 默认值：10
- 环境变量：（不支持）

## 实验性标志
```powershell
–experimental-corrupt-check-time
```
- 群集损坏检查通过之间的持续时间
- 默认值：0s
- 环境变量：ETCD_EXPERIMENTAL_CORRUPT_CHECK_TIME

```powershell
–experimental-compaction-batch-limit
```
- 设置每个压缩批处理中删除的最大修订。
- 默认值：1000
- 环境变量：ETCD_EXPERIMENTAL_COMPACTION_BATCH_LIMIT

```powershell
–experimental-peer-skip-client-san-verification
```
- 跳过客户端证书中对等连接的SAN字段验证。例如，如果群集成员在NAT后面的不同网络中运行，这可能会有所帮助。
在这种情况下，请务必使用采用基于私有证书颁发机构等证书--peer-cert-file，--peer-key-file，--peer-trusted-ca-file
- 默认值：false
- 环境变量：ETCD_EXPERIMENTAL_PEER_SKIP_CLIENT_SAN_VERIFICATION

[etcd配置官方](https://etcd.io/docs/v3.4.0/op-guide/configuration/)



