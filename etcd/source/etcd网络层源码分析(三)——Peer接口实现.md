通过上一节分析我们知道，etcdhttp.Transporter接口实现中多个方法是通过调用Peer实例的相应方法实现的，个人理解为etcdhttp.Transporter为网络传输的抽象。在分布式系统中的通信，其实就是节点之间的通信，以此细分出Peer节点的抽象。

## 结构体
```go
type Peer interface {
	//发送单个消息，该方法是非阻塞的，如果出现发送失败，则会将失败信息报告给底层的Raft接口
	send(m raftpb.Message)
	
	sendSnap(m snap.Message)//发送snap.Message,其他行为与上面的send()方法类似
	
	update(urls types.URLs)//更新对应节点暴露的URL地址
	
	//将指定的连接与当前的Peer绑定，Peer会将该连接作为Stream消息通道使用
	//当Peer不再使用该连接时，会将该连接关闭
	attachOutgoingConn(conn *outgoingConn)
	
	stop()//关闭当前Peer实例，会关闭底层的网络连接
}
```
```go
type peer struct {
	localID types.ID//本地当前的节点ID
	id types.ID//该peer实例对应的节点ID
	
	r Raft //Raft接口,在Raft接口实现的底层封装了etcd-raft模块。
	
	status *peerStatus //节点的状态
	picker *urlPicker//每个节点可能提供了多个URL供其他节点访问，当其中一个访问失败时，我们应该可以尝试访问另一个。而urlPicker提供的主要功能就是这些URL之间进行切换。
	
	msgAppV2Writer *streamWriter //v2版本负责向Stream消息通道写入消息
	writer         *streamWriter //v3版本负责向Stream消息通道写入消息
	pipeline       *pipeline //pipeline消息通道
	snapSender     *snapshotSender //负责发送快照数据
	msgAppV2Reader *streamReader//v2版本负责从Stream消息通道读取消息
	msgAppReader   *streamReader//v3版本负责从Stream消息通道读取消息
	
	recvc chan raftpb.Message //从Stream消息通道中读取消息之后，会通过该通道将消息交给Raft接口，然后由它返回给底层etcd-raft模块进行处理。
	propc chan raftpb.Message//从Stream消息通道中读取到MsgProp类型的消息之后，会通过该通道将MsgProp消息交给Raft接口，然后由它返回给底层etcd-raft模块进行处理。
	
	paused bool//是否暂停向对应的节点发送消息
}
```

## peer启动
在上一节中有提到过在[Transport](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/transport.go)的方法AddPeer会启动startPeer
```go
func startPeer(t *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats) *peer {
	...
	status := newPeerStatus(t.Logger, t.ID, peerID)//初始化peer状态
	picker := newURLPicker(urls)//根据节点的URL创建urlPicker
	errorc := t.ErrorC
	r := t.Raft  //底层的Raft状态机
	pipeline := &pipeline{ //创建pipeline实例
		peerID:        peerID,
		tr:            t,
		picker:        picker,
		status:        status,
		followerStats: fs,
		raft:          r,
		errorc:        errorc,
	}
	pipeline.start()//启动pipeline

	p := &peer{//创建Peer实例
		lg:             t.Logger,
		localID:        t.ID,
		id:             peerID,
		r:              r,
		status:         status,
		picker:         picker,
		msgAppV2Writer: startStreamWriter(t.Logger, t.ID, peerID, status, fs, r),//创建并启动v2版本的streamWriter
		writer:         startStreamWriter(t.Logger, t.ID, peerID, status, fs, r),//创建并启动v3版本的streamWriter
		pipeline:       pipeline,
		snapSender:     newSnapshotSender(t, picker, peerID, status),
		recvc:          make(chan raftpb.Message, recvBufSize),//创建recvc通道，注意缓冲区大小
		propc:          make(chan raftpb.Message, maxPendingProposals),//创建propc通道，注意缓冲区大小
		stopc:          make(chan struct{}),
	}

	ctx, cancel := context.WithCancel(context.Background())
	p.cancel = cancel
	/*
	启动单独的goroutine,它主要负责将recvc通道中读取消息，该通道中的消息就是从对端节点发送过来
	的消息，然后将读取到的消息交给底层的Raft状态机进行处理。
	*/
	go func() {
		for {
			select {
			case mm := <-p.recvc://从recvc通道中获取连接上读取到的Message
				//将Message交给底层Raft状态机处理
				if err := r.Process(ctx, mm); err != nil {
					if t.Logger != nil {
						t.Logger.Warn("failed to process Raft message", zap.Error(err))
					}
				}
			case <-p.stopc:
				return
			}
		}
	}()

	/*
	底层Raft状态机处理MsgProp类型的Message时，可能会阻塞，所以启动单独的goroutine来处理
	*/
	go func() {
		for {
			select {
			case mm := <-p.propc:
				if err := r.Process(ctx, mm); err != nil {//从propc通道中获取MsgProp类型的Message
					//将Message交给底层Raft状态机处理
					if t.Logger != nil {
						t.Logger.Warn("failed to process Raft message", zap.Error(err))
					}
				}
			case <-p.stopc:
				return
			}
		}
	}()
	
	//创建并启动v2版本的streamReader实例，它主要负责从Stream消息通道上读取消息
	p.msgAppV2Reader = &streamReader{
		lg:     t.Logger,
		peerID: peerID,
		typ:    streamTypeMsgAppV2,
		tr:     t,
		picker: picker,
		status: status,
		recvc:  p.recvc,
		propc:  p.propc,
		rl:     rate.NewLimiter(t.DialRetryFrequency, 1),
	}
	
	//创建并启动v3版本的streamReader实例，它主要负责从Stream消息通道上读取消息
	p.msgAppReader = &streamReader{
		lg:     t.Logger,
		peerID: peerID,
		typ:    streamTypeMessage,
		tr:     t,
		picker: picker,
		status: status,
		recvc:  p.recvc,
		propc:  p.propc,
		rl:     rate.NewLimiter(t.DialRetryFrequency, 1),
	}

	p.msgAppV2Reader.start()
	p.msgAppReader.start()
	...
}
```
rafthttp.peer.startPeer()方法在启动的时候会启动pipeline和stream的两种消息通道类型,pipeline是短连接的，stream则是HTTP长连接，所以stream需要专门负责写入数据的streamWriter和专门读取数据的streamReader。
最终pipeline和stream接收到的消息都会被写入到peer.recvc和peer.propc再由etcd-raft模块统一处理。

## send发送消息
上一节中[Transport](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/transport.go)的方法send就是通过调用peer.send方法来实现消息发送功能的。
```c
func (p *peer) send(m raftpb.Message) {
	p.mu.Lock()
	paused := p.paused//检测paused字段，是否暂停对指定节点发送消息
	p.mu.Unlock()

	if paused {
		return
	}
	//根据消息的类型选择合适的消息通道
	writec, name := p.pick(m)
	select {
	case writec <- m://根据消息的类型将Message写入writc通道中,等待发送
	default:
		//如果发送出现阻塞，则将信息报告给底层raft状态机，
		p.r.ReportUnreachable(m.To)
		if isMsgSnap(m) {
			p.r.ReportSnapshot(m.To, raft.SnapshotFailure)
		}
	}
}
```
在发送消息的时候，会根据消息的类型选择合适的消息通道，并返回相应的通道供send()方法写入待发送的消息,这个选择通道的方法是由peer.pick()方法来完成的。
```c
func (p *peer) pick(m raftpb.Message) (writec chan<- raftpb.Message, picked string) {
	var ok bool
	//如果是MsgSnap类型的消息，则返回Pipeline消息通道对应的Channel,否则返回Stream消息通道
	if isMsgSnap(m) {
		return p.pipeline.msgc, pipelineMsg
	} else if writec, ok = p.msgAppV2Writer.writec(); ok && isMsgApp(m) {
		return writec, streamAppV2
	} else if writec, ok = p.writer.writec(); ok {
		return writec, streamMsg
	}
	return p.pipeline.msgc, pipelineMsg
}
```
 1. 如果该消息是MsgSnap快照消息,则选择pipeline.msgc通道
 2. 如果消息类型是MsgApp并且v2版本的streamWriter处于工作状态,则选择v2版本的streamWriter.msgc通道
 3. 如果v3版本的streamWriter处于工作状态，则选择v3版本的streamWriter.msgc通道
 4. 最后，如果都不符合则直接使用pipeline.msgc通道


## attachOutgoingConn获得连接
前面提到stream消息通道类型是HTTP长连接的,rafthttp.peer.attachOutgoingConn作用就是独自对网络连接做了一层封装，供streamWriter进行绑定，绑定之后http处理器就可以将缓存区的数据刷到对端,这个方法是为stream消息通道实现的。
```c
func (p *peer) attachOutgoingConn(conn *outgoingConn) {
	var ok bool
	switch conn.t {
	case streamTypeMsgAppV2:
		ok = p.msgAppV2Writer.attach(conn)//v2版本streamWriter绑定连接
	case streamTypeMessage:
		ok = p.writer.attach(conn)//v3版本streamWriter绑定连接
	default:
		if p.lg != nil {
			p.lg.Panic("unknown stream type", zap.String("type", conn.t.String()))
		}
	}
	if !ok {
		conn.Close()
	}
}
```

更多欢迎关注[go成神之路](https://github.com/friendlyhank/toBeTopgopher)


