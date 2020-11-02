本文会对etcd网络层源码展开系列课程讲解，本篇文章主要目的是为了让读者更快摸索到网络层是如何建立连接和收发数据，后面以这个为切入口做更为细致的讲解。

etcd网络层源码主要在[etcd-rafthttp](https://github.com/etcd-io/etcd/tree/master/server/etcdserver/api/rafthttp)模块,etcd-rafthttp模块的实现,降低了etcd-raft模块与网络之间的耦合,降低了和etcd-raft模块的实现成本，提高整个程序的可扩展性。etcd-rafthttp模块的实现依靠etcd底层[网络组件](https://github.com/etcd-io/etcd/tree/master/pkg/transport),网络组件是对golang net网络库的具体实现。

## 节点之间网络拓扑结构
首先在etcd集群中，每个节点启动时都会与集群中的其他节点建立网络连接，这里以三个节点的集群为例，最终形成的网络结构：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020102911335377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70#pic_center)
## 节点之间消息通信
etcd接受对外客户端的连接，但是这里我们只讲节点之间的消息通信。
我们都知道，建立socket服务端一共有五个基本步骤(C语言):创建socket套接字、bind地址和端口、listen监听服务、accept接收客户端的连接、启动新线程为客户端服务。
套接字服务启动在[startEtcd](https://github.com/etcd-io/etcd/blob/master/server/etcdmain/etcd.go)方法中完成。节点之间即作为客户端也作为服务端。
```go
func StartEtcd(inCfg *Config) (e *Etcd, err error) {
	...
	//设置集群节点的监听服务
	if e.Peers, err = configurePeerListeners(cfg); err != nil {
		return e, err
	}
	//对外的服务端监听服务
	if e.sctxs, err = configureClientListeners(cfg); err != nil {
		return e, err
	}
	
	//集群节点accept接收客户端的连接、启动新线程为客户端服务
	if err = e.servePeers(); err != nil {
		return e, err
	}
	if err = e.serveClients(); err != nil {
		return e, err
	}
	...
}
```
configurePeerListeners中通过调用上面提到的底层网络组件，设置listen监听，节点和对外客户端服务的通信都支持tls安全验证。
```go
func configurePeerListeners(cfg *Config) (peers []*peerListener, err error) {
	...
	peers = make([]*peerListener, len(cfg.LPUrls))
	for i, u := range cfg.LPUrls {
		peers[i] = &peerListener{close: func(context.Context) error { return nil }}
		peers[i].Listener, err = rafthttp.NewListener(u, &cfg.PeerTLSInfo)
	}
	...
}
```

servePeers方法使用第三方库cmux来实现，cmux里有具体的accept接受客户端连接并启用线程为客户端服务，对这两个细节做了隐藏，同时因为etcd里支持grpc,所以servePeers方法参杂着grpc相关的网络服务,这里先不对grpc相关做讲解，先行忽略。etcd之间的通信主要是通过http来实现的,那么客户端的请求逻辑和响应逻辑就交由Handler完成。
```go
func (e *Etcd) servePeers() (err error) {
	
	ph := etcdhttp.NewPeerHandler(e.GetLogger(), e.Server)	
	for _, p := range e.Peers {
		m := cmux.New(p.Listener)
		srv := &http.Server{
			Handler:     grpcHandlerFunc(gs, ph),
			ReadTimeout: 5 * time.Minute,
			ErrorLog:    defaultLog.New(ioutil.Discard, "", 0), // do not log user error
		}
		go srv.Serve(m.Match(cmux.Any()))
		p.serve = func() error { return m.Serve() 
	}
	
	// start peer servers in a goroutine
	for _, pl := range e.Peers {
		go func(l *peerListener) {
			u := l.Addr().String()
			e.cfg.logger.Info(
				"serving peer traffic",
				zap.String("address", u),
			)
			e.errHandler(l.serve())
		}(pl)
	}
}
```

这里值得注意的是方法[NewPeerHandler](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/etcdhttp/peer.go)中etcdserver实现了四个Handler接口类型，每种对应不同的网络处理逻辑，因为节点之间通信最终服务的是具有高一致性的raft协议,所以我们主要关注的是RaftHandler()，RaftHandler接口的具体实现是交由etcd网络的核心transport去处理Handler的。
```c
func NewPeerHandler(lg *zap.Logger, s etcdserver.ServerPeerV2) http.Handler {
	return newPeerHandler(lg, s, s.RaftHandler(), s.LeaseHandler(), s.HashKVHandler(), s.DowngradeEnabledHandler())
}

func (s *EtcdServer) RaftHandler() http.Handler { return s.r.transport.Handler() }
```

讲到这里网络的基本过程已经讲完了，这时候etcd已经具备建立连接和处理响应的条件了，那么进一步我们来分析，etcd是如何发起连接，如何发送数据和接收数据的呢？

## etcd网络结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201029161150638.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70#pic_center)
 1. transporter接口作为etcd网络的核心，主要职责维持HTTP长连接，包含连接请求以及处理响应的Handler逻辑和提供发送消息的接口，以及包含对每个peer节点的维护操作，同时，也为etcd-raft模块架起沟通的桥梁。
 2. Peer接口是集群具体节点的抽象，在分布式系统中的通信说到底就是节点之间的通信。
 3. 在往下一层网络etcd针对不同的消息类型所携带的数据大小不同，分出两种不同的传输通道stream和pipeline，各自特点：
 		 - Stream类型通道：点到点之间维护HTTP长连接，主要传输数据量较小、比较频繁的的数据，例如追加日志、心跳等。
 		 - Pipeline类型通道：点到点之间短连接，用完即关闭，主要传输数据量较大、发送频率较低的消息，例如快照消息等。
 Stream消息通道是在节点启动后，主动与集群中的其他节点建立的，每个Stream消息通道有2个关联的后台goroutine,其中一个启动[streamReader](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)主动发起**dial**建立关联的HTTP连接，并从连接上读取数据，另外一个后台goroutine启动[streamWriter](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)在建立起连接之后，负责发送数据。
 Pipeline消息通道则是使用短连接，启用4个线程去发送数据。
 4.etcd网络层的底层依赖围绕golang net包来实现的。

最后，本文只是对etcd网络层的如何建立连接和收发数据的过程做大致的讲解，后面会针对性的做详细的分析。

更多欢迎关注[go成神之路](https://github.com/friendlyhank/toBeTopgopher)
