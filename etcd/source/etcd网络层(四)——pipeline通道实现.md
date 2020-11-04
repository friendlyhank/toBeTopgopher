pipeline作为网络传输的通道之一，主要负责传输快照数据。

## 结构体
```go
type pipeline struct {
	peerID types.ID //该pipeline对应的节点的ID
	tr     *Transport //关联rafthttp.Transport实例
	picker *urlPicker //用于选择可用的url
	status *peerStatus //节点的状态
	raft   Raft//底层raft实例
	msgc chan raftpb.Message//pipeline实例从该通道中获取待发送的消息
	wg    sync.WaitGroup//负责同步多个goroutine结束。每个pipeline默认开启4个goroutine来处理msgc中的消息，必须先关闭这些goroutine，才能真正关闭该pipeline
	stopc chan struct{}
}
```

## 工作原理
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201104082032837.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70#pic_center)
 1. pipeline在启动的时候会启动4个goroutine来发送消息
 2. rafthttp.peer.send()在发送消息的时候会选择合适的通道，进入待发送状态
 3. pipeline.handle()将pipeline.msgc通道接收到要发送的消息后,调用pipeline.post()将其发送出去
 4. rafthttp.Transport.Handler()方法pipelineHandler的Handler实现负责接收pipeline发送的数据，接受完后再将消息提交到etcd-raft模块。

## 启动
在上一节中[peer](https://github.com/etcd-io/etcd/blob/master/server/etcdserver/api/rafthttp/peer.go)的startPeer方法，有对pipeline的初始化和启动的操作。
```go
func (p *pipeline) start() {
	p.stopc = make(chan struct{})
	p.msgc = make(chan raftpb.Message, pipelineBufSize)//初始化msgc通道，默认缓冲是64个

	p.wg.Add(connPerPipeline)
	for i := 0; i < connPerPipeline; i++ {//默认开启4个goroutine来处理msgc中待发送的消息
		go p.handle()//发送消息
	}
}
```
pipeline.start()会做初始化和启动用来发送消息的后台goroutine。

## handle方法处理msgc中待发送的消息
pipeline.handle在上面的pipeline.start的时候使用到。
```go
func (p *pipeline) handle() {
	defer p.wg.Done()//handle()方法执行完成，也就是当前这个goroutine结束

	for {
		select {
		case m := <-p.msgc: //接收待发送的MsgSnap类型的消息
			start := time.Now()
			err := p.post(pbutil.MustMarshal(&m))//将消息序列化，然后创建HTTP请求并发送出去
			end := time.Now()

			if err != nil {
				//将通道的网络连接状态置为不活跃
				p.status.deactivate(failureType{source: pipelineMsg, action: "write"}, err.Error())

				if m.Type == raftpb.MsgApp && p.followerStats != nil {
					p.followerStats.Fail()
				}
				//向底层的Raft状态机报告失败信息
				p.raft.ReportUnreachable(m.To)	
				if isMsgSnap(m) {
					p.raft.ReportSnapshot(m.To, raft.SnapshotFailure)
				}
				sentFailures.WithLabelValues(types.ID(m.To).String()).Inc()
				continue
			}
			//发送成功,将通道的网络连接状态置为活跃
			p.status.activate()
			if m.Type == raftpb.MsgApp && p.followerStats != nil {
				p.followerStats.Succ(end.Sub(start))
			}
			//向底层的Raft状态机报告发送成功的消息
			if isMsgSnap(m) {
				p.raft.ReportSnapshot(m.To, raft.SnapshotFinish)
			}
			sentBytes.WithLabelValues(types.ID(m.To).String()).Add(float64(m.Size()))
		case <-p.stopc:
			return
		}
	}
}
```
在pipeline.handle()方法中会从msgc通道中读取待发送的Message消息，然后调用pipeline.post()方法将其发送出去，发送结束之后会调用底层Raft接口的相应方法报告发送结果。

## post发送消息
pipeline.post在上面的pipeline.handle的时候使用。
```go
func (p *pipeline) post(data []byte) (err error) {
	u := p.picker.pick()//选择可用的url
	//创建HTTP POST请求
	req := createPostRequest(p.tr.Logger, u, RaftPrefix, bytes.NewBuffer(data), "application/protobuf", p.tr.URLs, p.tr.ID, p.tr.ClusterID)

	done := make(chan struct{}, 1)//主要用于通知下面的goroutine请求是否已经发送完成
	ctx, cancel := context.WithCancel(context.Background())
	req = req.WithContext(ctx)
	go func() { //该goroutine主要用于监听请求是否取消
		select {
		case <-done:
		case <-p.stopc: //如果在请求得发送过程中,pipeline被关闭,则取消该请求
			waitSchedule()
			cancel()//取消请求
		}
	}()
	//发送HTTP POST请求,并获取到对应的响应。
	resp, err := p.tr.pipelineRt.RoundTrip(req)
	done <- struct{}{}//通知上述goroutine,请求已经发送完毕
	if err != nil {
		p.picker.unreachable(u)
		return err
	}
	defer resp.Body.Close()
	b, err := ioutil.ReadAll(resp.Body)//等到响应的结果
	if err != nil {
		p.picker.unreachable(u)//出现异常时，则将该URL标识为不可用
		return err
	}
	//检查响应的内容
	err = checkPostResponse(p.tr.Logger, resp, b, req, p.peerID)
	if err != nil {
		p.picker.unreachable(u)
		// errMemberRemoved is a critical error since a removed member should
		// always be stopped. So we use reportCriticalError to report it to errorc.
		if err == errMemberRemoved {
			reportCriticalError(err, p.errorc)
		}
		return err
	}

	return nil
}
```
pipeline.post()方法是真正完成消息发送的地方，其中会启动一个后台goroutine监听控制发送过程及获取发送结果。

更多欢迎关注[go成神之路](https://github.com/friendlyhank/toBeTopgopher)


