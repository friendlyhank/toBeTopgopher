Transporter接口在包rafthttp中作为网络层核心接口之一，包含连接请求以及处理响应的Handler逻辑和提供发送消息的接口，以及包含对每个peer节点的维护操作，同时，也为etcd-raft模块架起通讯的桥梁。

## 结构体
```go
type Raft interface {
	//将指定的消息实例传递到底层的etcd-raft模块进行处理
	Process(ctx context.Context, m raftpb.Message) error
	IsIDRemoved(id uint64) bool//检测指定节点是否从当前集群中移除
	ReportUnreachable(id uint64)//通知底层etcd-raft模块,当前节点与指定的节点无法连通
	ReportSnapshot(id uint64, status raft.SnapshotStatus)//通知底层etcd-raft模块,快照数据是否发送成功
}
```
```go
type Transporter interface {
	Start() error //初始化操作
	Handler() http.Handler//创建Handler实例，并关联到指定的URL上
	Send(m []raftpb.Message) //发送消息
	SendSnapshot(m snap.Message)//发送快照数据
	AddRemote(id types.ID, urls []string)//在集群中添加一个节点时，其他节点会通过该方法添加该新节点消息
	AddPeer(id types.ID, urls []string)//添加指定集群节点(自己除外)
	RemovePeer(id types.ID) //移除集群指定节点
	RemoveAllPeers() //移除所有的节点
	UpdatePeer(id types.ID, urls []string)//修改节点信息
	ActiveSince(id types.ID) time.Time //节点存活时长的统计
	ActivePeers() int //处于活跃状态得节点数
	Stop() //关闭所有连接
}
```
```go
type Transport struct {
	DialTimeout time.Duration //拨号超时的时间
	DialRetryFrequency rate.Limit //拨号重试的频率
	
	TLSInfo transport.TLSInfo //tls信息安全

	ID          types.ID //当前节点自己的ID
	URLs        types.URLs//当前节点与集群其他节点交互时使用的URL地址
	ClusterID   types.ID //当前节点所在的集群ID
	//Raft是一个接口，其实现的底层封装了etcd-raft模块，当rafthttp.Transport收到消息之后，会将其交给raft实例进行处理。
	Raft        Raft 
	Snapshotter *snap.Snapshotter //负责管理快照文件
	
	streamRt   http.RoundTripper //stream消息通道中使用http.RoundTripper,实现了http.Transport,用来完成一个HTTP事务
	pipelineRt http.RoundTripper//pipeline消息通道中使用http.RoundTripper,实现了http.Transport,用来完成一个HTTP事务
	remotes map[types.ID]*remote//remotes中只封装了pipeline实例，remote主要负责发送快照数据，帮助新加入的节点快速追赶其他节点的数据。
	peers   map[types.ID]Peer//Peer接口是当前节点对集群中其他节点的抽象表示。对于当前节点来说，集群中其他节点在本地都会有一个Peer实例与之对应，peers字段维护了节点ID到对应Peer实例之间的映射关系。

	pipelineProber probing.Prober//用于探测pipeline通道是否可用
	streamProber   probing.Prober //用于探测stream通道是否可用
}
```
rafthttp.Transport是rafthttp.Transporter接口具体实现，它提供了向对等端发送raft消息的功能,并且接受其他节点的消息。下面对Transport接口实现做进一步讲解。

## Transport初始化
transport初始化在方法[StartEtcd](https://github.com/etcd-io/etcd/blob/master/server/embed/etcd.go)启动etcd并在方法[NewServer](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/server.go)初始化etcdserver时候完成。

```go
func (t *Transport) Start() error {
	//创建Stream消息通道使用http.RoundTripper实例，底层实际上是创建http.Transport实例
	//这里创建连接的超时时间(根据配置指定),读写请求的超时时间(默认5s),keepAlive时间(默认30s)
	t.streamRt, err = newStreamRoundTripper(t.TLSInfo, t.DialTimeout)
	
	//创建Pipeline消息通道使用http.RoundTripper实例，底层实际上是创建http.Transport实例
	//创建连接的超时时间(根据配置指定),keepAlive时间(默认30s),与stream不同的是读写请求的超时时间设置为永不过期
	t.pipelineRt, err = NewRoundTripper(t.TLSInfo, t.DialTimeout)
	
	//初始化remotes和peers两个map字段
	t.remotes = make(map[types.ID]*remote)
	t.peers = make(map[types.ID]Peer)
	//初始化prober实例用于探测pipeline和stream消息是否可用
	t.pipelineProber = probing.NewProber(t.pipelineRt)
	t.streamProber = probing.NewProber(t.streamRt)
	
	if t.DialRetryFrequency == 0 {
		t.DialRetryFrequency = rate.Every(100 * time.Millisecond)//拨号频次限定间隔100毫秒
	}
}
```
这里重点要提一下的就是http.RoundTripper,在使用net.http库发送http请求过程中，最后都调用了http.Transport.RoundTrip()方法,http.Transport是RoundTripper接口实现之一。
实际上streamRt和pipelineRt底层就是http.Transport,在RoundTripper接口中，定义了RoundTrip()方法来完成一个HTTP事务，那么什么是HTTP事务呢？简单来说就是：客户端发送一个HTTP Request到服务端，服务端处理完请求后，返回相应的HTTP Respose的过程。

## Handler方法
在http库中,Handler接口主要是封装了处理客户端请求逻辑及创建返回响应的逻辑。
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

在上一章中提到过etcd网络在启动之初的Handler逻辑会转化到[etcdserver](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/server.go)实现的四个接口类型RaftHandler、LeaseHandler、HashKVHandler、DowngradeEnabledHandler中，其中RaftHandler接口的具体实现就是transport.Handler。
```go
func (s *EtcdServer) RaftHandler() http.Handler { return s.r.transport.Handler() }
```

transport.Handler主要负责创建Stream消息通道和Pipeline消息通道用到的Handler实例，并注册到相应的请求路径上。
```go
func (t *Transport) Handler() http.Handler {
	pipelineHandler := newPipelineHandler(t, t.Raft, t.ClusterID)
	streamHandler := newStreamHandler(t, t, t.Raft, t.ID, t.ClusterID)
	snapHandler := newSnapshotHandler(t, t.Raft, t.Snapshotter, t.ClusterID)
	mux := http.NewServeMux()
	mux.Handle(RaftPrefix, pipelineHandler)
	mux.Handle(RaftStreamPrefix+"/", streamHandler)
	mux.Handle(RaftSnapshotPrefix, snapHandler)
	mux.Handle(ProbingPrefix, probing.NewHandler())
	return mux
}
```
url为/raft，则用pipelineHandler处理收到的快照数据
url为/raft/stream,则用streamHandler处理收到的消息
url为/raft/snapshot,则用snapshotHandler处理收到的快照数据

### pipelineHandler
pipelineHandler的ServeHTTP方法的实现
```go
func (h *pipelineHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	//pipeline使用的是POST方法
	if r.Method != "POST" {
		w.Header().Set("Allow", "POST")
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		return
	}
	
	/*
	限制每次从底层连接读取的字节数上线，默认是64KB，因为快照数据可能非常大，为了防止读取超时，
	只能每次读取一部分数据到缓冲区中，最后将全部数据拼接起来，得到完整的快照数据
	*/
	limitedr := pioutil.NewLimitedBufferReader(r.Body, connReadLimitByte)
	b, err := ioutil.ReadAll(limitedr)
	
	//反序列化得到raftpb.Message
	var m raftpb.Message
	if err := m.Unmarshal(b); err != nil {
		h.lg.Warn(
			"failed to unmarshal Raft message",
			zap.String("local-member-id", h.localID.String()),
			zap.Error(err),
		)
		http.Error(w, "error unmarshalling raft message", http.StatusBadRequest)
		recvFailures.WithLabelValues(r.RemoteAddr).Inc()
		return
	}
	
	//最后将得到的raftpb.Message消息对象交由etcd-raft模块处理消息
	if err := h.r.Process(context.TODO(), m); err != nil {
		switch v := err.(type) {
		case writerToResponse:
			v.WriteTo(w)
		default:
			h.lg.Warn(
				"failed to process Raft message",
				zap.String("local-member-id", h.localID.String()),
				zap.Error(err),
			)
			http.Error(w, "error processing raft message", http.StatusInternalServerError)
			w.(http.Flusher).Flush()
			// disconnect the http stream
			panic(err)
		}
		return
	}
}
```

###  streamHandler
streamHandler的ServeHTTP方法的实现
```go
func (h *streamHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	//streamHandler用的是GET方法
	if r.Method != "GET" {
		w.Header().Set("Allow", "GET")
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		return
	}
	
	//根据不同的版本获取streamType
	var t streamType
	switch path.Dir(r.URL.Path) {
	case streamTypeMsgAppV2.endpoint(h.lg):
		t = streamTypeMsgAppV2 //v2版本
	case streamTypeMessage.endpoint(h.lg):
		t = streamTypeMessage //v3版本
	default:
		h.lg.Debug(
			"ignored unexpected streaming request path",
			zap.String("local-member-id", h.tr.ID.String()),
			zap.String("remote-peer-id-stream-handler", h.id.String()),
			zap.String("path", r.URL.Path),
		)
		http.Error(w, "invalid path", http.StatusNotFound)
		return
	}
	
	//获取对端节点的ID
	fromStr := path.Base(r.URL.Path)
	from, err := types.IDFromString(fromStr)
	
	//根据对端节点ID获取对应的Peer实例
	p := h.peerGetter.Get(from)
	
	w.WriteHeader(http.StatusOK)//返回成功状态码200
	w.(http.Flusher).Flush()  //调用Flush()方法将响应数据发送到对端节点

	c := newCloseNotifier()
	conn := &outgoingConn{//创建outgoingConn实例
		t:       t,
		Writer:  w,
		Flusher: w.(http.Flusher),
		Closer:  c,
		localID: h.tr.ID,
		peerID:  from,
	}
	p.attachOutgoingConn(conn)//建立连接，将outgoingConn实例与对应的streamWriter实例绑定
	<-c.closeNotify()//阻塞等待消息发送,如果不阻塞了说明HTTP长连接被关闭
}
```
由于stream通道是长连接，当获取到对端请求后，将连接封装成outgoingConn实例，有streamWriter发送心跳来维护长连接。并给对端响应状态码。
###  snapshotHandler
snapshotHandler的ServeHTTP方法的实现

```go
func (h *snapshotHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.Method != "POST" {
		w.Header().Set("Allow", "POST")
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		snapshotReceiveFailures.WithLabelValues(unknownSnapshotSender).Inc()
		return
	}
	
	/*
	限制读取缓冲区数据最大为1TB
	*/
	dec := &messageDecoder{r: r.Body}
	// let snapshots be very large since they can exceed 512MB for large installations
	m, err := dec.decodeLimit(snapshotLimitByte)
	from := types.ID(m.From).String()
	
	//持久化数据到硬盘
	n, err := h.snapshotter.SaveDBFrom(r.Body, m.Snapshot.Metadata.Index)
	if err != nil {
		msg := fmt.Sprintf("failed to save KV snapshot (%v)", err)
		h.lg.Warn(
			"failed to save incoming database snapshot",
			zap.String("local-member-id", h.localID.String()),
			zap.String("remote-snapshot-sender-id", from),
			zap.Uint64("incoming-snapshot-index", m.Snapshot.Metadata.Index),
			zap.Error(err),
		)
		http.Error(w, msg, http.StatusInternalServerError)
		snapshotReceiveFailures.WithLabelValues(from).Inc()
		return
	}
	
	//最后将得到的raftpb.Message消息对象交由etcd-raft模块处理消息
	if err := h.r.Process(context.TODO(), m); err != nil {
		switch v := err.(type) {
		// Process may return writerToResponse error when doing some
		// additional checks before calling raft.Node.Step.
		case writerToResponse:
			v.WriteTo(w)
		default:
			msg := fmt.Sprintf("failed to process raft message (%v)", err)
			h.lg.Warn(
				"failed to process Raft message",
				zap.String("local-member-id", h.localID.String()),
				zap.String("remote-snapshot-sender-id", from),
				zap.Error(err),
			)
			http.Error(w, msg, http.StatusInternalServerError)
			snapshotReceiveFailures.WithLabelValues(from).Inc()
		}
		return
	}
}
```

## AddPeer添加节点
添加节点的操作主要在方法[StartEtcd](https://github.com/etcd-io/etcd/blob/master/server/embed/etcd.go)启动etcd并在方法[NewServer](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/server.go)初始化etcdserver时候完成。

```go
func NewServer(cfg ServerConfig) (srv *EtcdServer, err error) {
	...
	for _, m := range remotes {
		if m.ID != id {
			tr.AddRemote(m.ID, m.PeerURLs)
		}
	}
	for _, m := range cl.Members() {
		if m.ID != id {
			tr.AddPeer(m.ID, m.PeerURLs)
		}
	}
	...
}
```
AddRemote针对的是新加入的节点，AddPeer则是集群中除去当前节点外其他节点的映射。

```go
func (t *Transport) AddPeer(id types.ID, us []string) {
	...
	urls, err := types.NewURLs(us)//解析us切片中指定URL连接
	//创建指定节点对应的Peer实例，其中会相关的Stream消息通道和Pipeline消息通道
	t.peers[id] = startPeer(t, urls, id, fs)
	//每隔一段时间，prober会向该节点发送探测消息，检查对端的健康状况
	addPeerToProber(t.Logger, t.pipelineProber, id.String(), us, RoundTripperNameSnapshot, rttSec)
	addPeerToProber(t.Logger, t.streamProber, id.String(), us, RoundTripperNameRaftMessage, rttSec)
	...
}
```

AddPeer方法的主要工作就是创建并启动对应节点的Peer实例。

## Send发送消息
上面提到transport的发送消息和接收消息其实都是围绕etcd-raft模块来实现的，接收消息我们在下一节的Peer做讲解，transport的发送消息在[EtcdServer](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/server.go)方法run启动的时候会启动[raftNode](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/raft.go)方法start
```go
(r *raftNode) start(rh *raftReadyHandler) {
	...
	go func() {
		for {
			select {
				case rd := <-r.Ready():
					if islead {
					// gofail: var raftBeforeLeaderSend struct{}
					r.transport.Send(r.processMessages(rd.Messages))
					}
					
					if !islead {
						msgs := r.processMessages(rd.Messages)
						r.transport.Send(msgs)
					}
			}
		}
	}
	...
}

```

Transport.Send()方法负责发送指定的raft.Message消息，其中首先尝试使用目标节点对应的Peer实例发送消息，如果没有找到对应的Peer实例，则尝试使用对应的remote实例发送消息。
```go
func (t *Transport) Send(msgs []raftpb.Message) {
	for _, m := range msgs {//遍历msgs切片中的全部消息
		//根据raftpb.Message.To字段,获取目标节点对应的Peer实例
		to := types.ID(m.To)

		t.mu.RLock()
		p, pok := t.peers[to]
		g, rok := t.remotes[to]
		t.mu.RUnlock()
		
		if pok {//如果存在对应的Peer实例，则使用Peer发送消息
			if m.Type == raftpb.MsgApp {//统计信息
				t.ServerStats.SendAppendReq(m.Size())
			}
			p.send(m)//通过peer.send()方法完成消息的发送
			continue
		}

		if rok {//如果指定节点ID不存在对应的Peer实例，则尝试使用查找对应的remote实例
			g.send(m)//通过remote.send()方法完成消息的发送
			continue
		}
	}
}
```

更多欢迎关注[go成神之路](https://github.com/friendlyhank/toBeTopgopher)

