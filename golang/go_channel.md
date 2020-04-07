```go
type hchan struct {
	dataqsiz uint           //缓冲槽大小
	buf      unsafe.Pointer//缓冲槽指针
	sendx    uint   // 发送索引
	recvx    uint   // 接受索引
	recvq    waitq  // 接收队列
	sendq    waitq  // 发送队列
}
```
异步和同步的区别，在于是否由缓冲槽。

## 发送
当我们想要向channel发送数据的时候,系统会调用runtime.chansend1,然后会调用runtime.chansend，这个函数负责了发送数据的全部逻辑，如果我们调用时将block参数设置为true,那么就表示当前发送操作是一个阻塞操作。
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

}
```
runtime.chansend主要分为三个部分：
 - 当存在等待的接收者时，通过runtime.send直接将数据发送给阻塞的接收者。
 - 当缓存区存在空余的空间时，将发送的数据写入channel的缓冲区。
 - 当不存在缓冲区或者缓存区已经满时，等待其他Goroutine从Channel接收数据。

**直接发送**
如果目标Channel没有被关闭并且由处于读等待的Goroutine,那么chansend函数会从接收队列recvq中取出最先陷入等待的Goroutine并直接向它发送数据。
```go
if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```
send的时候回去唤醒接收者
**缓冲区**
```go
if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
```

**阻塞发送**
当 Channel 没有接收者能够处理数据时，向 Channel 发送数据就会被下游阻塞，当然使用 select 关键字可以向 Channel 非阻塞地发送消息。向 Channel 阻塞地发送数据会执行下面的代码，我们可以简单梳理一下这段代码的逻辑：

```go
if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
```

## 接收
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
}
```
 - 当存在等待的发送者时，通过runtime.recv直接从阻塞的发送者或者缓存区中获取数据。
 - 当缓冲区存在数据时，从Channel的缓冲区接收数据。
 - 当缓冲区中不存在数据时，等待其他Goroutine 向 Channel 发送数据；

**直接接收**
```go
if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
```
```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if ep != nil {
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	gp := sg.g
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}
```
接收的时候也会去唤醒发送方。

该函数会根据缓冲区的大小分别处理不同的情况：

如果 Channel 不存在缓冲区；
 1. 调用 runtime.recvDirect 函数会将 Channel 发送队列中 Goroutine 存储的 elem 数据拷贝到目标内存地址中；
如果 Channel 存在缓冲区；
 1. 将队列中的数据拷贝到接收方的内存地址；
 2. 将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；

**缓存区**
当Channel 的缓冲区中已经包含数据时，从 Channel 中接收数据会直接从缓冲区中 recvx 的索引位置中取出数据进行处理：
```go
if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
```

**阻塞接收**
当 Channel 的发送队列中不存在等待的 Goroutine 并且缓冲区中也不存在任何数据时，从管道中接收数据的操作会变成阻塞操作，然而不是所有的接收操作都是阻塞的，与 select 语句结合使用时就可能会使用到非阻塞的接收操作：
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

## 关闭通道

