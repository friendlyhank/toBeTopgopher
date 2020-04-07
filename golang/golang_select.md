很多 C 语言或者 Unix 开发者听到 select 想到的都是系统调用，而谈到 I/O 模型时最终大都会提到基于 select、poll 和 epoll 等函数构建的 IO 多路复用模型。Go 语言的 select 与 C 语言中的 select 有着比较相似的功能。

select 可以管理**多个channel**的**收发**操作。

## 现象
当我们在 Go 语言中使用 select 控制结构时，会遇到两个有趣的现象：
 - select 能在 Channel 上进行非阻塞的收发操作；
 - select 在遇到多个 Channel 同时响应时会随机挑选 case 执行；
这两个现象是学习 select 时经常会遇到的，我们来深入了解具体的场景并分析这两个现象背后的设计原理。

## 非阻塞的收发
在通常情况下，select 语句会阻塞当前 Goroutine 并等待多个 Channel 中的一个达到可以收发的状态。但是如果 select 控制结构中包含 default 语句，那么这个 select 语句在执行时会遇到以下两种情况：
 1. 当存在可以收发的 Channel 时，直接处理该 Channel 对应的 case；
 2. 当不存在可以收发的 Channel 是，执行 default 中的语句

## 随机执行
```go
func main() {
	ch := make(chan int)
	go func() {
		for range time.Tick(1 * time.Second) {
			ch <- 0
		}
	}()

	for {
		select {
		case <-ch:
			println("case1")
		case <-ch:
			println("case2")
		}
	}
}
```
从上述代码输出的结果中我们可以看到，select 在遇到多个 <-ch 同时满足可读或者可写条件时会随机选择一个 case 执行其中的代码。

编译器在中间代码生成期间会根据 select 中 case 的不同对控制语句进行优化，这一过程都发生在 cmd/compile/internal/gc.walkselectcases 函数中，我们在这里会分四种情况介绍处理的过程和结果：

 1. select 不存在任何的 case；
 2. select 只存在一个 case；
 3. select 存在两个 case，其中一个 case 是 default；
 4. select 存在多个 case；


### 直接阻塞
首先介绍的是最简单的情况，也就是当 select 结构中不包含任何 case 时编译器是如何进行处理的，我们截取 cmd/compile/internal/gc.walkselectcases 函数的前几行代码：
```go
n := cases.Len()
	sellineno := lineno

	// optimization: zero-case select
	if n == 0 {
		return []*Node{mkcall("block", nil, nil)}
	}
```
这段代码非常简单并且容易理解，它直接将类似 select {} 的空语句转换成调用 runtime.block 函数：
```go
func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1) // forever
}
```
runtime.block 函数的实现非常简单，它会调用 runtime.gopark 让出当前 Goroutine 对处理器的使用权，传入的等待原因是 waitReasonSelectNoCases。

简单总结一下，空的 select 语句会直接阻塞当前的 Goroutine，导致 Goroutine 进入无法被唤醒的永久休眠状态。
```go
select{}
```

### 单一管道
如果当前的 select 条件只包含一个 case，那么就会将 select 改写成 if 条件语句。下面展示了原始的 select 语句和被改写、优化后的代码：
```go
// 改写前
select {
case v, ok <-ch: // case ch <- v
    ...    
}

// 改写后
if ch == nil {
    block()
}
v, ok := <-ch // case ch <- v
...
```
cmd/compile/internal/gc.walkselectcases 在处理单操作 select 语句时，会根据 Channel 的收发情况生成不同的语句。当 case 中的 Channel 是空指针时，就会直接挂起当前 Goroutine 并永久休眠。

## 非阻塞操作
当 select 中仅包含两个 case，并且其中一个是 default 时，Go 语言的编译器就会认为这是一次非阻塞的收发操作。cmd/compile/internal/gc.walkselectcases 函数会对这种情况单独处理，不过在正式优化之前，该函数会将 case 中的所有 Channel 都转换成指向 Channel 的地址。我们会分别介绍非阻塞发送和非阻塞接收时，编译器进行的不同优化。
