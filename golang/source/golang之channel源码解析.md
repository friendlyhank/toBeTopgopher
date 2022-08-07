> 不要通过共享内存的方式通信，而是应该通过通信的方式共享内存
> 

![在这里插入图片描述](https://img-blog.csdnimg.cn/318d5591178d48d79e369f32694c9779.png)

在大多数编程语言的并发模型中，在传递数据的时候一般是通过共享内存的方式，这种方式存在线程之间的竞态。而go里面Goroutine之间可以通过Channel传输数据，Channel本身也用到锁资源，但两者粒度不一样，两者都是并发安全的。

## 结构体
![在这里插入图片描述](https://img-blog.csdnimg.cn/c81c3799c2ed4bea80e8f3078fe016a6.png)
channel分为发送者sendq和接收者recvq，在没有缓冲槽情况，发送端和接收端都可能发生阻塞的状态。有缓冲槽的的情况则根据缓冲槽是否满了，来决定阻塞和非阻塞。因为发送和接收本身可以理解为是解耦的,channel相当于是队列，即FIFO，遵循的是先进先出的概念。

## 初始化
```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem
	
	// 计算所需的空间的大小elem.size*size,并判断是否溢出
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0: // 没有缓冲槽，size是0,分配chan内存空间
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 当前的chan和缓冲槽分配一块连续的内存空间
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 指针单独为chan和缓冲槽分配内存空间
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	return c
}
```

## 发送
发送可以分为三个部分:

1.有接收者，将数据写到接受者内存地址，并唤醒对应接收者的g。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0329d7f7ff334e478ff27a1f18a1dc48.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/e3543ac0b76c4ccf8aaa53c69440719c.png)

2.没有接收者，但是有缓冲区

![在这里插入图片描述](https://img-blog.csdnimg.cn/23bfe4ac58e242f2bae1ea50fad41aec.png)

3.没有接收者，没有缓冲区或缓冲区已满，将数据写入sudog，在发送者队列尾部插入,g进入休眠状态
![在这里插入图片描述](https://img-blog.csdnimg.cn/aa2d86b8428d4a02ac3c08609c59c564.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/86557cce5e074ca7b60df53189901c33.png)

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	//有等待接收的队列,直接将数据发送到接收端的内存地址中,并设置对应的g为可执行状态
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	//chan有缓存槽,把数据写入缓冲槽中
	if c.qcount < c.dataqsiz {
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

	if !block {
		unlock(&c.lock)
		return false
	}
	
	//如果没有缓存槽,往发送队列中加入一条等待执行的sudgo
	gp := getg()
	//获取一个等待执行的sudgo
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	//将数据写入sugo
	mysg.elem = ep
	mysg.waitlink = nil
	//sudo绑定goroutione
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	
	atomic.Store8(&gp.parkingOnChan, 1)
	//让当前goroutine进入休眠状态,等待被唤醒
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

	KeepAlive(ep)

	//到这里说明gorourtine被接收端唤醒，对sugo资源进行释放
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 将数据拷贝到对端
	if sg.elem != nil {
		// 拷贝值到目标内存地址
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg) // g也会绑定sudog信息
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 唤醒处于读等待的Goroutine
	goready(gp, skip+1)
}
```

```go
func goready(gp *g, traceskip int) {
	// 系统栈执行调度
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}
```

```go
func ready(gp *g, traceskip int, next bool) {
    // 获取g的状态
	status := readgstatus(gp)
	// g0
	_g_ := getg()
	// 获取g0上的m
	mp := acquirem()
	if status&^_Gscan != _Gwaiting {
		dumpgstatus(gp)
		throw("bad g->status in ready")
	}
	// 修改g的状态为可执行状态
	casgstatus(gp, _Gwaiting, _Grunnable)
	// 放入p的优先者队列
	runqput(_g_.m.p.ptr(), gp, next)
	// 开始调度
	wakep()
	releasem(mp)
}
```

## 接收
接收也可以分为三部分:
1.有发送者队列
(1)无缓冲区产生了发送者队列

![在这里插入图片描述](https://img-blog.csdnimg.cn/93f38ebd663943478fc0b41a65360c31.png)

(2)有缓冲区(满了)且产生了发送者队列
![在这里插入图片描述](https://img-blog.csdnimg.cn/4caaab58440c4a4c86cb5af7c51f1283.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/6718d2eb10034cdb8b0c742b42d1ead6.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/7c80a7660bce4ce690ec789cd944c35a.png)

2.有缓冲区，没有发送者队列
![在这里插入图片描述](https://img-blog.csdnimg.cn/f0ef5b07f7bd412184a2065529726310.png)


3.没有发送者队列,没有缓冲区或缓冲区没数据，将接收数据指针地址写入sudog，在接收者队列尾部插入
![在这里插入图片描述](https://img-blog.csdnimg.cn/b6fac807f5d04fb6a7c57994e8b8e295.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/cae8e974cb3f43f9b4fe5018bccdc1bc.png)

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	lock(&c.lock)

	//如果有等待的发送队列,那么直接接收
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	//没有在等待的发送队列,缓冲槽中有数据，从缓存槽中读取数据
	if c.qcount > 0 {
		qp := chanbuf(c, c.recvx)
		
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		//将已经被拷贝到发送方的缓冲槽释放
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--//chan元素个数减少
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	//如果没有缓存槽,往接收队列中加入一条等待执行的sugo
	gp := getg()
	//获取一个等待执行的sugo
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	//将对应接受者指针地址写入sugo
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
	
	//让当前goroutine进入休眠状态,等待被唤醒
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
	
	//到这里说明gorourtine被发送端唤醒，对sugo资源进行释放
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)

	return true, success
}
```

```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 如果没有缓存槽,那么直接拷贝发送队列的值
	if c.dataqsiz == 0 {
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 如果有缓冲槽，说明缓存槽满了并且产生了等待发送的队列
		// 从缓冲槽中获取数据,并且将发送队列的头节点保存的数据写入缓冲槽中
		qp := chanbuf(c, c.recvx)
		// copy data from queue to receiver
		// 将缓冲槽的数据拷贝到接收者目标地址
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		//拷贝发送者数据到缓冲区
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++ // 接收索引增加
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// 缓冲槽满了的情况是缓冲槽发送索引等于接收的索引值
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 将阻塞的发送方头节点goroutine唤醒
	goready(gp, skip+1)
}
```

##  关闭
```go
func closechan(c *hchan) {
	c.closed = 1
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
		gp.param = unsafe.Pointer(sg)
		sg.success = false
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
		gp.param = unsafe.Pointer(sg)
		sg.success = false
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

