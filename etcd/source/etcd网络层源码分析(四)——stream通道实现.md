stream作为网络传输的通道之一，通过建立HTTP长连接，主要传输数据量小、发送频率较高的数据,例如追加日志、心跳数据。

## 工作原理
结合上一节的[pipeline通道实现](https://editor.csdn.net/md/?articleId=109329856)的工作原理，我们来看下stream的工作原理：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201105080041591.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70#pic_center)
我们知道stream是HTTP长连接，所以stream需要专门负责写入数据的streamWriter和专门读取数据的streamReader。

**streamReader工作原理：**
在[rafthttp.peer.startPeer()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/peer.go)中通过初始化并调用[streamReader.start()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)启动streamReader实例，其中还启动了一个后台goroutine来执行[streamReader.run()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)方法。在streamReader.run()方法中,主要完成下面两件事情：
 1. 通过调用[streamReader.dial()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)主动与对端建立连接
 2. 接收对端发送的数据,写入streamReader.recvc或streamReader.propc,这两个通道其实对应的是上层peer.recvc和peer.propc通道,最后消息在[rafthttp.peer.startPeer()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/peer.go)
中被接收并提交到etcd-raft模块被处理。

**streamWriter工作原理：**
在[rafthttp.peer.startPeer()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/peer.go)中通过调用[startStreamWriter()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)初始化并启动streamWriter实例，其中还启动了一个后台goroutine来执行[streamWriter.run()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go)方法。在streamWriter.run()方法中,主要完成下面三件事情:
1. 在[peer章节](https://blog.csdn.net/m0_37731056/article/details/109329808)中提到过当其他节点主动与当前节点创建连接([streamReader.dial()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/stream.go))时,该连接实例会写入对应streamWriter.connc通道,在streamWriter.run方法中通过该通道获取该连接实例并进行绑定，之后才能开始后续的消息发送和心跳检测。
2. 获取连接之后，定时将心跳消息写入缓冲区，通过HTTP处理器将缓冲区数据刷到对端,用于定期检测网络状况
3. 获取连接之后，从streamWriter.msgc通道接收到要发送的消息,将消息写入缓冲区,通过HTTP处理器将缓冲区数据刷到对端。

	
## streamWriter
### 结构体
```go
type streamWriter struct {
	localID types.ID//本地节点ID
	peerID  types.ID//对端节点ID
	
	status *peerStatus//网络连接状态
	r      Raft //底层的Raft状态
	
	mu      sync.Mutex // guard field working and closer
	closer  io.Closer//负责关闭底层的长连接
	working bool //负责标识当前的streamWriter是否可用
	msgc  chan raftpb.Message//通过前面对Peer分析可知,Peer会将待发送的消息写入该通道,streamWriter则从该通道中读取消息并发送出去
	connc chan *outgoingConn//通过该通道获取当前streamWriter实例关联的底层网络连接，  outgoingConn其实是对网络连接的一层封装，其中记录了当前连接使用的协议版本，以及用于关闭连接的Flusher和Closer等信息。
	stopc chan struct{}
	done  chan struct{}
}
```

### streamWriter启动
```go
func (cw *streamWriter) run() {
	var (
		msgc       chan raftpb.Message //指向当前streamWriter.msgc字段
		/**
		定时器会定时向该通道发送信号,触发心跳消息的发送,该心跳消息与后面介绍的Raft的心跳消息有所不同
		该心跳消息的主要目的是为了防止连接长时间不用断开的
		**/
		heartbeatc <-chan time.Time
		t          streamType//区分不同版本的stream消息通道
		enc        encoder//编码器,负责将消息序列化并写入连接的缓冲区
		flusher    http.Flusher//负责刷新底层连接,将数据真正发送出去
		batched    int //记录当前未Flush的消息个数
	)
	tickc := time.NewTicker(ConnReadTimeout / 3)//发送心跳消息的定时器
	defer tickc.Stop()
	unflushed := 0//未Flush的字节数

	for {
		select {
		case <-heartbeatc:
			//将心跳消息编码到缓冲区
			err := enc.encode(&linkHeartbeatMessage)
			unflushed += linkHeartbeatMessage.Size()//统计未发送的字节数
			if err == nil {
				//若没有异常,则使用flusher将缓冲区的消息全部发送出去,并重置batched和unflushed
				flusher.Flush()
				batched = 0
				sentBytes.WithLabelValues(cw.peerID.String()).Add(float64(unflushed))
				unflushed = 0
				continue //不阻塞,直接continue
			}
			//到这里说明有错误，将网络连接状态置为不活跃
			cw.status.deactivate(failureType{source: t.String(), action: "heartbeat"}, err.Error())
			
			sentFailures.WithLabelValues(cw.peerID.String()).Inc()
			cw.close()//关闭streamWriter,会导致底层连接的关闭
			if cw.lg != nil {
				cw.lg.Warn(
					"lost TCP streaming connection with remote peer",
					zap.String("stream-writer-type", t.String()),
					zap.String("local-member-id", cw.localID.String()),
					zap.String("remote-peer-id", cw.peerID.String()),
				)
			}
			heartbeatc, msgc = nil, nil

		case m := <-msgc://peer向streamWriter.msgc写入待发送的消息
			err := enc.encode(&m)//将消息编码到缓冲区
			if err == nil {
				unflushed += m.Size()
				//msgc通道中的消息全部发送完成或是未Flush得消息过多,则触发Flush
				//否则只是递增batched变量
				if len(msgc) == 0 || batched > streamBufSize/2 {
					flusher.Flush()
					sentBytes.WithLabelValues(cw.peerID.String()).Add(float64(unflushed))
					unflushed = 0
					batched = 0
				} else {
					batched++
				}
				continue
			}
			//到这里说明有错误，将网络连接状态置为不活跃(略)
		
		
		/**
		当其他节点(对端)主动与当前节点创建Stream消息通道时，会先通过StreamHandler的处理，
        StreamHandler会通过attach()方法将连接写入对应的peer.writer.connc通道，
        而当前的goroutine会通过该通道获取连接，然后开始发送消息
		**/
		case conn := <-cw.connc://获取当前streamWriter实例绑定的底层连接
			cw.mu.Lock()
			closed := cw.closeUnlocked()
			t = conn.t //获取该连接底层发送得消息版本,并创建相应的encoder实例
			switch conn.t {//这里只关注最新版本,忽略其他版本
			case streamTypeMsgAppV2:
				enc = newMsgAppV2Encoder(conn.Writer, cw.fs)
			case streamTypeMessage:
				//将http.ResponseWriter封装成messageEncoder,上层调用通过messageEncoder实例完成消息发送
				enc = &messageEncoder{w: conn.Writer}
			default:
			}
			flusher = conn.Flusher //记录底层连接对应的Flusher
			unflushed = 0 //重置未Flush得字节数
			cw.status.activate() //节点网络连接置为活跃状态
			cw.closer = conn.Closer//
			cw.working = true //标识当前streamWriter正在运行
			cw.mu.Unlock()

			//更新heartbeatc和msg两个通道,获取连接后才能发送消息
			heartbeatc, msgc = tickc.C, cw.msgc

		case <-cw.stopc:
			close(cw.done)
			return
		}
	}
}
```

### streamWriter绑定连接
在[rafthttp.peer.attachOutgoingConn()](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/peer.go)提到会调用streamWriter.attach()方法streamWriter与connc进行绑定
```go
func (cw *streamWriter) attach(conn *outgoingConn) bool {
	select {
	case cw.connc <- conn://将conn实例写入streamWriter.connc通道
		return true
	case <-cw.done:
		return false
	}
}
```
主要用于接收对端得连接并进行绑定。

## streamReader
###  结构体
```go
type streamReader struct {
	peerID types.ID //对应节点得ID
	typ    streamType//区分不同版本的stream消息通道

	tr     *Transport//关联rafthttp.Transport实例
	picker *urlPicker//用于获取对端可用的URL
	status *peerStatus//节点连接状态
	recvc  chan<- raftpb.Message////创建streamReader时会启动一个后台goroutine从peer.recvc通道中读取消息。从对端节点发送来的非MsgProp类型消息会写入到该通道中，然后由新建的goroutine读取出来，交给底层的etcd-raft模块进行处理
	propc  chan<- raftpb.Message//功能基本同上，只不过用于接收MsgProp类型的消息。

	rl *rate.Limiter //拨号得频率限制

	errorc chan<- error

	mu     sync.Mutex
	paused bool //是否暂停读取数据
	closer io.Closer

	ctx    context.Context
	cancel context.CancelFunc
	done   chan struct{}
}
```
###  streamReader启动
```go
func (cr *streamReader) run() {
	t := cr.typ //获取使用的消息版本

	for {
		//向对端节点发送一个GET请求，然后获取并返回响应的ReadCloser，主要用于和对端建立连接 
		rc, err := cr.dial(t)
		if err != nil {
			//发生错误，将连接状态置为不活跃
			if err != errUnsupportedStreamType {
				cr.status.deactivate(failureType{source: t.String(), action: "dial"}, err.Error())
			}
		} else {
			//成功建立连接,将连接状态置为活跃
			cr.status.activate()
			//读取对端返回的消息，并将读取到的消息写入streamReader.recvc通道中
			err = cr.decodeLoop(rc, t)
			
			switch {
			// all data is read out
			case err == io.EOF:
			// connection is closed by the remote
			case transport.IsClosedConnError(err):
			default:
				cr.status.deactivate(failureType{source: t.String(), action: "read"}, err.Error())
			}
		}
		//来到这里说明连接发送错误,等待一会,发起新的dial连接尝试,调用rate.Limiter可限制拨号频率
		err = cr.rl.Wait(cr.ctx)
	}
}
```

streamReader.dial()方法主要负责与对端节点建立连接。
```go
func (cr *streamReader) dial(t streamType) (io.ReadCloser, error) {
	u := cr.picker.pick()//选用对端可用的URL
	uu := u
	//根据使用协议得版本和节点ID创建最终得URL地址
	uu.Path = path.Join(t.endpoint(cr.lg), cr.tr.ID.String())

	//创建一个GET请求
	req, err := http.NewRequest("GET", uu.String(), nil)
	
	//设置头部信息
	req.Header.Set("X-Server-From", cr.tr.ID.String())
	req.Header.Set("X-Server-Version", version.Version)
	req.Header.Set("X-Min-Cluster-Version", version.MinClusterVersion)
	req.Header.Set("X-Etcd-Cluster-ID", cr.tr.ClusterID.String())
	req.Header.Set("X-Raft-To", cr.peerID.String())
	
	//将当前节点暴露得URL也一起发送给对端节点
	setPeerURLsHeader(req, cr.tr.URLs)
	
	//检测当前streamReader是否被关闭,若已被关闭，则返回异常
	req = req.WithContext(cr.ctx)

	cr.mu.Lock()
	select {
	case <-cr.ctx.Done():
		cr.mu.Unlock()
		return nil, fmt.Errorf("stream reader is stopped")
	default:
	}
	cr.mu.Unlock()
	//通过前面介绍的rafthttp.Transport.streamRt与对端建立连接
	resp, err := cr.tr.streamRt.RoundTrip(req)
	if err != nil {
		cr.picker.unreachable(u)
		return nil, err
	}

	//解析HTTP响应,检测版本信息
}
```
streamReader.decodeLoop()方法是streamReader得核心方法，它从底层得网络连接读取数据并进行反序列化，之后将得到得消息实例写入recvc或propc通道中,等待Peer进行处理。
```go
func (cr *streamReader) decodeLoop(rc io.ReadCloser, t streamType) error {
	var dec decoder
	cr.mu.Lock()
	//根据使用的协议版本创建对应的decoder实例,这里只关注协议的最新版本
	switch t {
	case streamTypeMsgAppV2:
		dec = newMsgAppV2Decoder(rc, cr.tr.ID, cr.peerID)
	case streamTypeMessage:
		dec = &messageDecoder{r: rc}//初始化messageDecoder实例并从连接中得到数据
	default:
	}
	//检测streamReader是否关闭,关闭则返回异常
	select {
	case <-cr.ctx.Done():
		cr.mu.Unlock()
		if err := rc.Close(); err != nil {
			return err
		}
		return io.EOF
	default:
		cr.closer = rc
	}
	cr.mu.Unlock()

	for {
		//从底层连接中读取数据,并反序列化成raftpb.message实例
		m, err := dec.decode()

		cr.mu.Lock()
		paused := cr.paused
		cr.mu.Unlock()
		
		if paused {//检测是否已经暂停从该连接上读取数据
			continue
		}
		//忽略连接层的心跳消息,注意与后面介绍的raft得心跳消息进行区分
		if isLinkHeartbeatMessage(&m) {
			// raft is not interested in link layer
			// heartbeat message, so we should ignore
			// it.
			continue
		}
		//根据读取到得消息类型,选择对应的通道进行写入
		recvc := cr.recvc
		if m.Type == raftpb.MsgProp {
			recvc = cr.propc
		}

		select {
		case recvc <- m://将消息写入对应的通道中,之后会交给底层Raft状态机进行处理
		default:
		}
	}
}
```

更多欢迎关注[go成神之路](https://github.com/friendlyhank/toBeTopgopher)


