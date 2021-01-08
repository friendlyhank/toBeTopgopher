在工作日常中，我们常常需要将对象转化为指定格式的数据或将指定格式的数据转化为对象,最常见得如:json、protobuf。在转化过程中，开发者因为定义字段等习惯上的不同，字段可能包含特殊字符或大小写等问题,本身go语言是对大小写敏感的，导致在转化对象过程产生问题,为了解决这个问题,struct tag就是在转化过程中提供与struct之间建立映射关系方便转化。
struct tag应用广泛，最常见的如json、protobuf、数据库映射orm,xorm等。

## 格式
```go
type strutName struct{
   fieldName type `key:"value,opt1,opt2,opts..." key2:"value2,opt1,opt2,opts..."`
}
```
 - 结构体字段fieldName首字母必须是大写,否则会被忽略(在go中首字母大小写表示公私有属性)。
 - tag都是以key:"value"的形式存在,多个键值对之间用空格隔开，好处是可以构建多个json解析器
 - value与opt1之间用','分割,第一个参数必须是value,value非必填,不填写时默认为fieldName
 - 参数opt为string,可以将bool、int、int8、int16、int32、int64、uint、uint8、uint16、uint32、uint64、uintptr、float32、float64转化为string类型。
 - 参数opt为omitempty,如果字段值为空则忽略字段
 
## 用法
### 结构体字段大小写
结构体字段首字母必须大写，否则会因为公私有属性问题，字段无法编解码。
```go
package main

import (
	"encoding/json"
	"fmt"
)

type UserInfo struct{
	name string
	age int `json:"age"`
	address string `json:"address"`
}

func main(){
	user := &UserInfo{name: "张三",age:18,address: "广东天河"}

	b,_ := json.Marshal(user)
	fmt.Println(string(b))
}
```

```go
{}
```

### 多json编码解析器
现在市面上出现越来越多的json序列化框架,如fastjson、gson、jackjson、protobuf,每种框架效率可能不尽相同,有时候在系统中就会出现多种序列化编码解析器，多个解析器之间以空格分开。下面以json、protobuf为例:
```go
type UserInfo struct {
	Name    string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
	Age     int32  `protobuf:"varint,2,opt,name=age" json:"age,omitempty"`
	Address string `protobuf:"bytes,3,opt,name=address" json:"address,omitempty"`
}

func main(){
	user := &pb.UserInfo{Address: "广东天河"}

	//json编码
	b,_ := json.Marshal(user)
	fmt.Println(string(b))

	//protobuf编码
	p,_ := proto.Marshal(user)
	fmt.Println(string(p))
}
```

### 忽略空值
omitempty表示在编解码时如果字段值为空，则忽略这个字段(Defined as false, 0, a nil pointer, a nil interface value, and any empty array, slice, map, or string)
```go
type UserInfo struct{
	Name string `json:"name,omitempty"`
	Age int `json:"age,omitempty"`
	Address string `json:"address,omitempty"`
}

func main(){
	user := &UserInfo{Name: "张三",Age:18}
	b,_ := json.Marshal(user)
	fmt.Println(string(b))
}
```
```go
{"name":"张三","age":18}
```
### 忽略字段
'-'参数表示在编解码过程不管字段是否为空，都直接忽略这个字段
```go
type UserInfo struct{
	Name string `json:"name,omitempty"`
	Age int `json:"age,omitempty"`
	Address string `json:"-"`
}

func main(){
	user := &UserInfo{Name: "张三",Age:18,Address: "广东省广州市"}
	b,_ := json.Marshal(user)
	fmt.Println(string(b))
}
```
```go
{"name":"张三","age":18}
```
### 自定义
key可以自定义,value可以不需要,但是要逗号隔开
```go
type UserInfo struct{
	Name string `mytag:"name,omitempty"`
	Age int `mytag:"age,omitempty"`
	Address string `mytag:",omitempty"`
}
```

### 如何获取键值对的tag
```go
type UserInfo struct{
	Name string `json:"name,omitempty"`
	Age int `json:"age,omitempty"`
	Address string `json:"address,omitempty"`
}

func main() {
	var u UserInfo
	t := reflect.TypeOf(u)

	for i :=0;i<t.NumField();i++{
		sf :=t.Field(i)
		fmt.Println(sf.Tag.Get("json"))
	}
}
```

