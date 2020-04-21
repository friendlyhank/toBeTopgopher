## 概述
map是常见的一种数据结构,大部分编程语言都有,用于存储一系列无序的键值对,map也被称为字典或关联数组,顾名思义，键相当于索引,通过键与值形成映射关系,达到快速查找数据的目的。

## 声明和初始化
```go
var map[Key-Type]Value-Type
```
首先我们看看声明map的格式，Key-Type表示的是key的类型,一定要是comparable的,即是可以任意用==或者!=比较的类型,比如float、int、string、bool。切片,结构体,map是不能作为key的,但是指针和接口类型是允许的。

value是可以支持任意的类型,甚至是内置的map。

比如声明一个key为string类型,value为int类型的map:
```go
var mapScore map[string]int
```
在未初始化之前,map为nil,所以要进行初始化。


常用的初始化有make和花括号的形式2种方式:
方式一:
```go
mapScore  = make(map[string]int)
```
方式二:

```go
//初始化空的map
m = map[string]int{}

//初始化不为空的map
mapScore :=map[string]int{
		"Tom":78,
		"Mary":60,
		"Kevin":90,
	}
```
在初始化的时候,可以无需知道map的长度,map是会动态的扩容的。

## 取值
```go
score,exist := mapScore["Tom"]
if exist{
	fmt.Println("Name:Tom,Score:",score)
}
```
map取值会返回两个形参,第一个是根据value的类型,返回缺省值,第二个是bool类型,tue表示键值对存在,false表示不存在。

## 删除
```go
delete(mapScore,"Tom")
```

## 遍历
```go
for key, value := range mapScore {
		fmt.Println("Key:", key, "Value:", value)
	}
```
```go
Key: Mary Value: 60
Key: Kevin Value: 90
Key: Tom Value: 78
```
从遍历的结果可以看出输出的顺序是无法保证的。

## 排序：将无序变有序
上面提到过map是无序的,那怎么把map变成有序的呢?下面例子可以介绍在使用中也可能常用到的配合有序的切片slice将无序变成有序的过程。
比如需要按名字的英文字母排序输出:
```go
func main(){
	var names []string
	mapScore :=map[string]int{
		"Tom":78,
		"Mary":60,
		"Kevin":90,
	}

	for key, _ := range mapScore {
		names = append(names,key)
	}
	sort.Strings(names)
	for _,name:=range names{
		fmt.Println("Key:", name, "Value:", mapScore[name])
	}
}
```
```go
Key: Kevin Value: 90
Key: Mary Value: 60
Key: Tom Value: 78
```

## 函数传递：引用类型
```go
func main(){
	mapScore :=map[string]int{
		"Tom":78,
		"Mary":60,
		"Kevin":90,
	}
	fmt.Println(mapScore["Tom"])
	modify(mapScore)
	fmt.Println(mapScore["Tom"])
}
//修改
func modify(mapScore map[string]int){
	mapScore["Tom"]=66
}
```
运行结果是可以看出key为"Tom"的值被修改了,说明map是引用类型。




## 线程安全:不安全
map不是线程安全的,在读写的时候,往往需要加锁:
```go
type Accumulator struct{
	sync.RWMutex
	Accum map[string]int
}
```
读取数据时候,加上读锁
```go
func (a *Accumulator)Read(i int){
	a.RLock()
	defer  a.RUnlock()
	count :=a.Accum["goodsView"]
	fmt.Println("key:goodsView",count)
}
```
写数据时候,加上写锁
```go
func (a *Accumulator)Write(i int){
	a.Lock()
	defer a.Unlock()
	a.Accum["goodsView"]++
}
```

