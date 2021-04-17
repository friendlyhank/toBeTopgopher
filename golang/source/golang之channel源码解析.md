> 不要通过共享内存的方式通信，而是应该通过通信的方式共享内存

在go官方描述Channel这一特性的时候有提到,在大多数编程语言的并发模型中，在传递数据的时候一般是通过共享内存的方式，这种方式存在线程之间的竞态，而go语言中使用的是并发模型是Goroutine和Channel,Goroutine作为Go最小的执行单元，而Goroutine之间可以通过Channel传输数据,Channel本身也用到锁资源，是并发安全的。
## 结构体
```go
type hchan struct {
	qcount  uint     //元素的个数
	dataqsiz uint           //缓冲槽大小
	buf      unsafe.Pointer//缓冲槽指针
	elemsize uint16  //元素的大小
	closed uint32   //是否关闭状态
	elemtype *_type //元素的类型 element type
	sendx    uint   // 发送索引
	recvx    uint   // 接受索引
	
	recvq    waitq  // 等待接收goroutine队列列表 
	sendq    waitq  // 等待发送的goroutine队列列表
	
	lock mutex
}
```
channel分为发送者sendq和接收者recvq，在发送和接收时候都会有阻塞的状态，没有缓冲槽则会出现阻塞的情况，也是我们说的同步的概念，有缓冲槽的的情况则根据缓冲槽是否满了，来决定阻塞和非阻塞，非阻塞则是我们说的异步概念，因为发送和接收本身可以理解为是解耦的，另外从结构体我们可以看出channel本身也是有锁的，所以是线程安全的。
## 初始化
```bash
func makechan(t *chantype,size int)*hchan{
	elem := t.elem
	
	//计算需要的内存空间
	mem,overflow :=math.MulUintptr(elem.size,uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
	}

	var c *hchan
	switch  {
	case mem == 0: //无缓冲区
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0://如果不是指针类型
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem,nil,true))
		c.buf = add(unsafe.Pointer(c),hchanSize)
	default:
		//如果是指针
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem,elem,true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz =uint(size)
	return c
}
```
初始化chan比较简单，就是根据元素的数据类型和大小计算出需要的内存空间,再根据是否有缓存区数据来初始化chan和chan的buf缓存区。

channel相当于是队列，即FIFO，遵循的是先进先出的概念。
## 发送
当我们想要向channel发送数据的时候,系统会调用runtime.chansend1,然后会调用runtime.chansend，这个函数负责了发送数据的全部逻辑，如果我们调用时将block参数设置为true,那么就表示当前发送操作是一个阻塞操作。
```bash
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
```
runtime.chansend主要分为三个部分：

 1. 当存在等待的接收者时，通过runtime.send直接将数据发送给阻塞的接收者。

```bash
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	.....
	//有等待接收的队列,直接将数据发送到接收端的内存地址中
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	.....
```
### 直接发送
如果目标Channel没有被关闭并且由处于读等待的Goroutine,那么chansend函数会从接收队列recvq中取出最先进入等待的Goroutine直接向它发送数据并将读等待的Goroutine唤醒。
```bash
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	.....
	//直接发送数据到接收端
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	//唤醒处于读等待的Goroutine
	goready(gp, skip+1)
}
```

```bash
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)

	//直接将数据写入接收端的内存地址
	memmove(dst, src, t.size)
}
```

 2. 当缓存区存在空余的空间时，将发送的数据写入channel的缓冲区。
 ### 缓冲区
```bash
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	.....
	//chan有缓存槽,把数据写入缓冲槽中
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		//根据发送的索引计算出要写入缓冲槽的位置
		qp := chanbuf(c, c.sendx)
		
		//将数据拷贝到缓冲槽对应的位置
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
	.....
}
```

 3. 当不存在缓冲区或者缓存区已经满时，等待其他Goroutine从Channel接收数据。
### 阻塞发送
当 Channel 没有接收者能够处理数据时，向 Channel 发送数据就会被下游阻塞，当然使用 select 关键字可以向 Channel 非阻塞地发送消息。向 Channel 阻塞地发送数据会执行下面的代码，我们可以简单梳理一下这段代码的逻辑：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	//如果没有缓存槽,往发送队列中加入一条等待执行的sugo
	// Block on the channel. Some receiver will complete our operation for us.
	//获取当前运行的goroutine
	gp := getg()
	//获取一个等待执行的sugo
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	//将数据写入sugo
	mysg.elem = ep
	mysg.waitlink = nil
	//sudo绑定goroutione
	mysg.g = gp
	mysg.isSelect = false
	//sudo绑定chan
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	//将sugo加入chan的发送队列
	c.sendq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
	//让当前goroutine进入休眠状态,等待被唤醒
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	//保持ep的活跃状态
	KeepAlive(ep)
	
	//到这里说明gorourtine被接收端唤醒，对sugo资源进行释放
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
}
```
阻塞发送这里可以看到用到了sugo对象，包括阻塞接收的时候也会用的，实际上sugo就是把当时使用的goroutine和需要发送或接收的数据进行封装(当然这个goroutine是休眠状态的)，然后放到接收队列和发送队列之中，唤醒的时候也是拿到sugo对应的goroutine进行唤醒操作。
## 接收
 1. 当存在等待的发送者时，通过runtime.recv直接从阻塞的发送者或者缓存区中获取数据。
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	.....
	//如果有等待的发送队列,那么直接接收
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	.....
}
```

### 直接接收
```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	//如果没有缓存槽,那么直接拷贝发送队列的值
	if c.dataqsiz == 0 {
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		//如果有缓冲槽，说明缓存槽满了并且产生了等待发送的队列
		//从缓冲槽中获取数据,并且将发送队列的头节点保存的数据写入缓冲槽中
		qp := chanbuf(c, c.recvx)
		
		//将缓冲槽的数据拷贝到接收者目标地址
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		//将发送队列的头节点拷贝到当前缓存槽位置
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++ //接收索引增加
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		//缓冲槽满了的情况是缓冲槽发送索引等于接收的索引值
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	//将阻塞的发送方头节点goroutine唤醒
	goready(gp, skip+1)
}
```

该函数会根据缓冲区的大小分别处理不同的情况：

如果 Channel 不存在缓冲区；
(1)调用 runtime.recvDirect 函数会将 Channel 发送队列中 Goroutine 存储的 elem 数据拷贝到目标内存地址中；
如果 Channel 存在缓冲区；
(2)将队列中的数据拷贝到接收方的内存地址；
(3)将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；

 2. 当缓冲区存在数据时，从Channel的缓冲区接收数据。
### 缓冲区
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	.....
	//没有在等待的发送队列,缓冲槽中有数据，从缓存槽中读取数据
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		////将缓冲槽的数据拷贝到接收方目标地址
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		//将已经被拷贝到发送方的缓冲槽释放
		typedmemclr(c.elemtype, qp)
		c.recvx++//接收索引增加
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--//chan元素个数减少
		unlock(&c.lock)
		return true, true
	}
	.....
}
```

 3. 当缓冲区中不存在数据时，等待其他Goroutine 向 Channel 发送数据；
接收的时候也会去唤醒发送方。

## 阻塞接收
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	.....
	//如果没有缓存槽,往接收队列中加入一条等待执行的sugo
	
	//获取当前运行的goroutine
	gp := getg()
	//获取一个等待执行的sugo
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	
	//将数据写入sugo
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	//sudo绑定goroutione
	mysg.g = gp
	mysg.isSelect = false
	//sudo绑定chan
	mysg.c = c
	gp.param = nil
	//将sugo加入chan的接收队列
	c.recvq.enqueue(mysg)
	
	atomic.Store8(&gp.parkingOnChan, 1)
	//让当前goroutine进入休眠状态,等待被唤醒
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
	
	//到这里说明gorourtine被发送端唤醒，对sugo资源进行释放
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
```go
func closechan(c *hchan) {
	.....
	//声明一个g列表
	var glist gList

	//释放所有接收者队列，把等待的接收者队列的g加入到g列表中
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	
	//释放所有发送者队列，把等待的发送者队列的g加入到g列表中
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	//将所有阻塞的发送者和接收者的goroutine唤醒
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

