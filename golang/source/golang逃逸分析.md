首先我们先要了解一下
**什么是逃逸分析？**
在维基百科中有这样说明:
> 在编译程序优化理论中，逃逸分析是一种确定指针动态范围的方法——分析在程序的哪些地方可以访问到指针。它涉及到指针分析和形状分析。

**为什么要了解逃逸分析?**
我们需要了解在编程语言中内存的分配主要是分为堆和栈，虽然堆和栈的分配主要在编译过程由编译器帮我们自动完成，但是了解逃逸分析，对于内存的分配和释放原理，还有在使用过程到底是用值类型还是引用类型是非常有帮助的。

## 堆和栈
在了解逃逸分析之前，我们需要了解什么是堆和栈？
在我理解，堆和栈是内存存储的一种结构
 - 堆(heap) 用于动态分配内存，由程序员申请分配和释放，在go里面内存是自动回收的，有自己的一套垃圾回收机制(gc)，gc的好处在于不必过多关注于内存的管理引发的一系列内存泄漏的问题，但不好也在于gc也伴随着性能的消耗。
 - 栈(stack)栈是一种线性的存储结构,栈区的内存一般由编译器分配和释放，一般存储着函数的入参以及局部变量，这些参数和局部变量会随着整个函数生命周期结束而被销毁。

栈在分配和回收内存的开销很低，只需要2个CPU指令:PUSH和POP,而堆方面一个很大的开销在于垃圾回收。

## 逃逸分析的例子
**什么时候会发生逃逸？**
继续引用维基百科的说明：
> 如果一个子程序分配一个对象并返回一个该对象的指针，该对象可能在程序中被访问到的地方无法确定——这样指针就成功“逃逸”了。如果指针存储在全局变量或者其它数据结构中，因为全局变量是可以在当前子程序之外访问的，此时指针也发生了逃逸。

**go逃逸分析命令**
```go
go run -gcflags "-m -l" (-m打印逃逸分析信息，-l禁止内联编译)
```

### 需要空间过大逃逸
```go
package main

func MakeSlice() {
	s := make([]int, 9000, 9000)
	s[0] = 1
}

func main() {
	MakeSlice()
}
```
```go
$ go run -gcflags "-m -l" main.go
# command-line-arguments
.\main.go:4:11: make([]int, 9000, 9000) escapes to heap
```

### 指针逃逸
当局部变量在函数作为指针返回,指针可能被外部引用的时候会发生逃逸.
```go
package main

type User struct{
	name string
}

func GetUser(name string)*User{
	return &User{name: name}
}

func main() {
	user := GetUser("张三")
	user.name = "李四"
}
```
```go
$ go run -gcflags "-m -l" main.go
# command-line-arguments
.\main.go:7:14: leaking param: name
.\main.go:8:9: &User literal escapes to heap
```

### 动态类型逃逸
```go
package main

type User struct{
	name string
}

func GetUser(name string)interface{}{
	return User{name: name}
}

func main() {
	var name = "张三"
	GetUser(name)
}
```

### 闭包逃逸
```go
package main

func Increase() func() int {
	n := 0
	return func() int {
		n++
		return n
	}
}

func main() {
	in := Increase()
	in() // 1
	in() // 2
}
```

