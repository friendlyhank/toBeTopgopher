## 发展史
Go 的包管理方式是逐渐演进的，在之前,不管是内部依赖还是外部依赖,所有的依赖的包都是放在GOPATH中, 所引发的问题是：
 - 在引用时候如果依赖包做了修改，删除，外部更新，可能引入破坏性的错误。
 - 在生产环境中，也可能出现与测试环境运行不一致的问题。
 - 多个应用引用不同版本的包依赖也可能出问题。

针对上述问题，从1.5开始go提供了新特性Vendor,基本思路是将包放在当前工程的vendor目录下面，在编译的时候就会优先从vendor目录下寻找依赖包。你细品，每个人工作的语言环境千变万化，现在在项目中统一使用某个版本依赖包，是不是很香？不用再苦恼下载什么版本的包。其实,vendor新特性,依然是有利有弊，问题：
 - 依然无法对外部依赖包做到很好的版本控制,会给包的升级产生很大的问题，无法评估升级带来的风险。
 - 导致拷贝泛滥,某个包在不同的项目中各有一份copy，而且其版本可能不一样；当依赖的包比较多的时候，vendor目录也会非常庞大。

包管理是判定一门语言是否强大重要的因素。喜大普奔,从Go1.11开始加入了Go Module作为带有版本控制新的包管理形式。

## 环境配置
在设置Go Module之前，需要先设置下环境变量：
**GO111MODULE设置**
 - GO111MODULE=off，go命令行将不会支持module功能，寻找依赖包的方式将会沿用旧版本那种通过vendor目录或者GOPATH模式来查找。
 - GO111MODULE=on，go命令行会使用modules，不会去GOPATH目录下查找。
 - GO111MODULE=auto，默认值，go命令行将会根据当前目录是否包含go.mod来决定是否启用 Go modules。
 
开启方式:
```
go env -w GO111MODULE=on
```

**代理设置**
go可以设置国内代理地址去快速下载依赖包
 - https://goproxy.cn
 - https://goproxy.io
 - https://mirrors.aliyun.com/goproxy/ 
 
设置方式    
```go
go env -w GOPROXY=https://goproxy.cn,direct
```

## 创建一个新模块
本文在GOPATH目录外创建mygin项目，以导入外部依赖和内部依赖进行演示，实际上go mod兼容了GOPATH目录内的使用。

使用go mod int {模块路径}进行初始化，生成go.mod文件。
```go
go mod init hank.com/mygin 
```
可以看到mygin目录下生成了go.mod文件
```go
module hank.com/mygin

go 1.14
```

外部依赖很简单，直接写入路径即可。如果要导入的是内部依赖，那么就是直接go mod init初始化时的模块路径+包名
mygin
├── apis
│   └── apis.go
└── main.go

apis.go
```go
package apis

import "github.com/gin-gonic/gin"

func Ping(c *gin.Context) {
	c.JSON(200, gin.H{
		"message": "pong",
	})
}
```
main.go
```go
package main

import (
	"github.com/gin-gonic/gin"
	"hank.com/mygin/apis" //内部依赖
)

func main() {
	r := gin.Default()
	r.GET("/ping",apis.Ping)
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

初始化完成之后,只要执行go 构建命令，go test、go build、go run,就可以自动添加新的依赖关系。
```go
go run main.go

go: finding module for package github.com/gin-gonic/gin
go: downloading github.com/gin-gonic/gin v1.6.2
go: found github.com/gin-gonic/gin in github.com/gin-gonic/gin v1.6.2
go: downloading github.com/mattn/go-isatty v0.0.12
go: downloading gopkg.in/yaml.v2 v2.2.8
go: downloading github.com/gin-contrib/sse v0.1.0
go: downloading github.com/go-playground/validator/v10 v10.2.0
go: downloading github.com/golang/protobuf v1.3.3
go: downloading github.com/ugorji/go v1.1.7
go: downloading github.com/leodido/go-urn v1.2.0
go: downloading github.com/go-playground/universal-translator v0.17.0
go: downloading github.com/ugorji/go/codec v1.1.7
go: downloading github.com/go-playground/locales v0.13.0
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
```

这时候我们在来看看go.mod文件,多了导入外部依赖包的路径，还带上版本的信息，而内部依赖是不需要写入的。
```go
module hank.com/mygin

go 1.14

require github.com/gin-gonic/gin v1.6.2 // indirect
```

go.mod 是启用了 Go moduels 的项目所必须的最重要的文件，它描述了当前项目（也就是当前模块）的元信息，每一行都以一个动词开头，目前有以下 5 个动词:
 - module：用于定义当前项目的模块路径。
 - go：用于设置预期的 Go 版本。
 - require：用于设置一个特定的模块版本。
 - exclude：用于从使用中排除一个特定的模块版本。
 - replace：用于将一个模块版本替换为另外一个模块版本。

注意这时候下载的依赖包不再GOPATH目录下,而是放在src/pkg/mod目录下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200411152229474.png)
除了go mod文件，再目录下还会生成go sum文件，加密hash,防止被恶意篡改。

## 升级依赖
**go get升级**：
```go
go get github.com/gin-gonic/gin
```
go get 默认值为@latest,会自动升级最新稳定版本的依赖包,除此之外go get还有多种方式：
```go
//指定版本
go get github.com/gin-gonic/gin@v1.6.2
//指定分支
go get github.com/gin-gonic/gin@master
//指定git commit id
go get github.com/gin-gonic/gin@e3702bed2
```

## 删除未使用的依赖
```go
go mod tidy
```

## 其他常用命令

**go list -m all查看所有依赖包**
添加一个依赖有可能带来其他依赖包的引入，查看所有依赖包选项：
```go
$ go list -m all

hank.com/mygin
github.com/davecgh/go-spew v1.1.1
github.com/gin-contrib/sse v0.1.0
github.com/gin-gonic/gin v1.6.2
github.com/go-playground/assert/v2 v2.0.1
github.com/go-playground/locales v0.13.0
github.com/go-playground/universal-translator v0.17.0
github.com/go-playground/validator/v10 v10.2.0
github.com/golang/protobuf v1.3.3
github.com/google/gofuzz v1.0.0
github.com/json-iterator/go v1.1.9
github.com/leodido/go-urn v1.2.0
github.com/mattn/go-isatty v0.0.12
github.com/modern-go/concurrent v0.0.0-20180228061459-e0a39a4cb421
github.com/modern-go/reflect2 v0.0.0-20180701023420-4b7aa43c6742
github.com/pmezard/go-difflib v1.0.0
github.com/stretchr/objx v0.1.0
github.com/stretchr/testify v1.4.0
github.com/ugorji/go v1.1.7
github.com/ugorji/go/codec v1.1.7
golang.org/x/sys v0.0.0-20200116001909-b77594299b42
golang.org/x/text v0.3.2
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
gopkg.in/check.v1 v0.0.0-20161208181325-20d25e280405
gopkg.in/yaml.v2 v2.2.8
```

**go list -m -versions列出依赖包可用的版本**
```go
go list -m -versions github.com/gin-gonic/gin

github.com/gin-gonic/gin v1.1.1 v1.1.2 v1.1.3 v1.1.4 v1.3.0 v1.4.0 v1.5.0 v1.6.0 v1.6.1 v1.6.2
```

参考:
[https://github.com/golang/go/wiki/Modules](https://github.com/golang/go/wiki/Modules)
[https://blog.golang.org/using-go-modules](https://blog.golang.org/using-go-modules)

更多系统学习欢迎关注github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)





