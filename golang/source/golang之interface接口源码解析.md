接口的定义可以说是一种规范，是一组方法的集合，通常在代码设计的层面，对多个组件有共性的方法进行抽象(共性可以分为横向和纵向)引入一层中间层，解除上游与下游的耦合关系，让代码可读性更高并不用关心方法的具体实现，同时借助接口也可以实现多态。

共性可以分为横向和纵向的例如动物这个对象可以向下细分为狗和猫，它们有共同的行为可以跑，这个为纵向。
再或者数据库的连接可以抽象为接口，可以支持mysql、oracle等，这个为横向。

## 结构体
interface的定义在1.15.3源码包runtime中,interface的定义分为两种，一种是不带方法的runtime.eface和带方法的runtime.iface。先来分别看看这两个结构体：

runtime.eface表示不含方法的interface{}类型,结构体包含可以表示任意数据类型的_type和存储指定的数据data,data用指针来表示。
```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```
```go
type _type struct {
	size       uintptr //占用的字节大小
	ptrdata uintptr //指针数据 size of memory prefix holding all pointers
	hash       uint32 //计算的hash
	tflag      tflag //额外的标记信息
	align      uint8 //内存对齐系数
	fieldAlign uint8 //字段内存对齐系数
	kind uint8 //用于标记数据类型
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool//用于判断当前类型多个对象是否相等
	str       nameOff //名字偏移量
	ptrToThis typeOff //指针的偏移量
}
```
runtime.iface表示包含方法的接口,结构体包含itab和data数据,itab包含的是接口类型interfacetype和装载实体的任意类型_type以及实现接口的方法fun,fun是可变大小,go在编译期间就会对接口实现校验检查,并将对应的方法存储fun。
```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype //接口类型的表示
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
不含方法的interface可以用来表示任意的数据类型,但是请注意interface本身也是一种数据类型(interface类型)。

## 类型转换
我们先来看一段代码
```go
type Print interface {
	Print()
}

type User struct {
	Name string
	Age int
}

func (r User) Print()  {
	fmt.Printf("hello %v,Age %v",r.Name,r.Age)
}

func main() {
	//eface
	var y interface{} = User{}
	fmt.Println(y)

	//iface
	var p Print= User{Name: "小明",Age: 18}
	p.Print()
}
```
分别设置了一个不包含方法的接口类型y和带方法的接口p，我们来看看汇编
```go
go build -gcflags '-l' -o main main.go
go tool objdump -s "main\.main" main
```

```go
//eface
CALL runtime.convT2E(SB)    //调用convT2E将User{}转化成eface

//iface
CALL    runtime.convT2I(SB) //将User实例和itab转化成iface
```

```go
func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
	//根据元素类型申请一块内存
	x := mallocgc(t.size, t, true)
	//将数据拷贝的分配的内存中
	typedmemmove(t, x, elem)
	//初始化eface
	e._type = t
	e.data = x
	return
}
```

```go
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
	t := tab._type
	//根据元素类型申请一块内存
	x := mallocgc(t.size, t, true)
	typedmemmove(t, x, elem)
	//将数据拷贝的分配的内存中
	//初始化iface
	i.tab = tab
	i.data = x
	return
}
```
在编译期间,go会将interface类型、对应接口方法、实例的类型、实例对应的方法内存地址组装成itab对象,最后在将data数据一起组装成iface对象。go对简单的类型有做优化可以不做转换，对一些较为复杂的类型也会根据不同的类型调用不同的方法转换，但其实都是大同小异。

## 动态派发
动态派发是在运行期间选择具体多态操作(方法或者函数)执行的过程，它是面向对象语言中的常见特性。
```go
func main() {
	var p Print= User{Name: "小明",Age: 18}
	p.Print()
}
```
先来看看第一种编译的结果：
```go
LEAQ    go.itab."".User,"".Print(SB), AX //将User实例和Print接口绑定成itab放入AX
CALL    runtime.convT2I(SB) //将User实例和itab转化成iface
MOVQ    AX, "".p+32(SP)       //iface.tab = SP+32 = AX
MOVQ    CX, "".p+40(SP)       //iface.data = SP + 40 = CX

MOVQ    "".p+32(SP), AX  //AX =iface.tab
MOVQ    24(AX), AX       //AX =iface.tab.fun[0] = User.Print()
MOVQ    "".p+40(SP), CX  //CX = iface.data
CALL    AX   //调用User实例Print方法
```
通过上述编译可以看出，编译器先将User转换成接口iface,在转换中进入到动态派发过程，最终确定数据类型的方法tab.fun和数据tab.data,然后进行方法的调用。

## 断言


