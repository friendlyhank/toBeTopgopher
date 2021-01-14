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

## 临时重定义tag
有时候在编码过程中,因为数据差异等问题，我们可以临时重新定义tag，重定义几乎包含tag的操作,其中包括:忽略空字段、添加字段、忽略字段、甚至拆分成多个或合并结构体等。

下面以忽略空字段举例,其他情况基本类似。在原结构体中address未添加忽略空字段的tag,但在序列化的时候重新定义address的标签为:当address信息未填写的时候，序列化时忽略该字段。
```go
package main

import (
	"encoding/json"
	"fmt"
)

type UserInfo struct{
	Name string `json:"name"`
	Age int `json:"age,string"`
	Address string `json:"address"`
}

func main(){
	user := UserInfo{Name: "张三",Age:18}
	b,_ := json.Marshal(struct {
		UserInfo
		Address string `json:"address,omitempty"`
	}{
		UserInfo:user,
	})
	fmt.Println(string(b))
}
```
```go
{"name":"张三","age":"18"}
```
这种用法使用场景较少，会损耗一些性能，如果tag标签重定义的比较散乱，会引起未知错误的风险，维护起来也非常麻烦，除了特殊情况，一般不会使用。

## json.Number
(1)如果接收的不知道是string、float64、int64,则可以用到json.Number,这种情况比较少,只是想说明json.Number在输入和输出转化这三种数据的时候都是非常方便的。
```go
package main

import (
	"encoding/json"
	"fmt"
)

type UserInfo struct{
	Name string `json:"name"`
	Age int `json:"age"`
	Address string `json:"address"`
}

func main(){
	str := `{"name":"张三","age":18,"address":"广东省广州市天河区"}`
	user := &UserInfo{}
	err := json.Unmarshal([]byte(str),user)
	if err != nil{
		panic(err)
	}
	fmt.Printf("%+v",user)

	str1 := `{"name":"张三","age":"18","address":"广东省广州市天河区"}`
	user1 := &UserInfo{}
	err = json.Unmarshal([]byte(str1),user1)
	if err != nil{
		print(err)
	}
	fmt.Printf("%+v",user1)
}
```

age给到的是string类型，无法解析成我们想要的数据:
```go
&{Name:张三 Age:18 Address:广东省广州市天河区}
&{Name:张三 Age:0 Address:广东省广州市天河区}
```

改进:
注意我们只需把age int改成json.Number的内置类型
```go
type UserInfo struct{
	Name string `json:"name"`
	Age json.Number `json:"age"`
	Address string `json:"address"`
}
```

(2)对于未定义的数据类型
用于interface类型编解码过程数值会被默认解析成float64类型，如果数值较大会精度丢失的问题
```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	s := `{"idno":9223372036854775807}`

	d := make(map[string]interface{})
	err := json.Unmarshal([]byte(s), &d)
	if err != nil {
		panic(err)
	}

	s2, err := json.Marshal(d)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(s2))
}
```

改进方法:
```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

func main() {
	s := `{"idno":9223372036854775807}`
	decoder := json.NewDecoder(strings.NewReader(s))
	decoder.UseNumber()
	d := make(map[string]interface{})
	err := decoder.Decode(&d)
	if err != nil{
		panic(err)
	}

	s2,err := json.Marshal(d)
	if err != nil{
		panic(err)
	}
	fmt.Println(string(s2))
}
```

## json.RawMessage
有些json数据是根据按照某些字段去解析的，这时候就要先接收最原始的编解码数据，然后再做进一步的解析,根据官方案例：
```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

func main() {
	type Color struct {
		Space string
		Point json.RawMessage // delay parsing until we know the color space
	}
	type RGB struct {
		R uint8
		G uint8
		B uint8
	}
	type YCbCr struct {
		Y  uint8
		Cb int8
		Cr int8
	}

	var j = []byte(`[
	{"Space": "YCbCr", "Point": {"Y": 255, "Cb": 0, "Cr": -10}},
	{"Space": "RGB",   "Point": {"R": 98, "G": 218, "B": 255}}
]`)
	var colors []Color
	err := json.Unmarshal(j, &colors)
	if err != nil {
		log.Fatalln("error:", err)
	}

	for _, c := range colors {
		var dst interface{}
		switch c.Space {
		case "RGB":
			dst = new(RGB)
		case "YCbCr":
			dst = new(YCbCr)
		}
		err := json.Unmarshal(c.Point, dst)
		if err != nil {
			log.Fatalln("error:", err)
		}
		fmt.Println(c.Space, dst)
	}
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

