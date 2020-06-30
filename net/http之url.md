## URL语法
```powershell
<scheme>://<user>:<password>@<host>:<port>/<path>;<params>?<query>#<frag>
```
 - 方案(scheme)：访问服务器以获取资源时要使用哪种协议
 - 用户(user)：某些方案使用资源时需要的用户名
 - 密码(pawwrod)：用户名后面可能要包含的密码，中间由冒号(:)分隔
 - 主机(host)：资源宿主服务器的主机名或者点分IP地址
 - 路径：服务器上资源的本地名，由一个斜杠(/)将其与前面URL组件分割开来。
 - 参数：某些方案会用这个组件来指定输入参数，参数为名/值对，URL中可以包含多个参数字段，它们互相之间以及与路径的其余部分之间用分号(:)分隔
 - 查询：某些方案会用这个组件来传递参数以激活应用程序，用?将其与URL的其余部分分割开来
 - 片段：一小片或一部分资源的名字。引用对象时,不会将frag字段传送给服务器，这个字段是客户端内部使用的。通过字段"#"将其与URL的其余部分分隔开来。

## 方案
方案实际上是规定如何指定资源的主要标识符，它会告诉负责解析URL的应用程序应该使用什么协议。在我们这个简单的HTTP URL中所使用的方案就是http。

## 主机与端口
要想在因特网上找到资源，应用程序要知道哪台机器装载了资源。
端口组件标识了服务器正在监听的网络端口，对下层使用了TCP协议的HTTP来说，默认端口为80。

第一个URL是通过主机名，第二个是通过IP地址指向服务器的：
 - https://www.baidu.com/
 - http://161.58.228.45:80/index.html

## 用户名和密码
很多服务器要求输入用户名和密码才会允许用户访问数据。FTP服务器就是这样一个常见的实例。

## 路径
URL的路径组件说明了资源位于服务器的什么地方。路径通常很像一个分级文件系统路径。

## 参数
为了向应用程序提供它们所需的输入参数，以便正确地与服务器进行交互，URL有一个参数组件。这个组件就是URL中的键值对列表，由字符";"将其与URL的其余部分(以及各键值对)分隔开来。
```bash
http://www.joes-hardware.com/hammers;sale=false/index.html;graphics=true
```
这个例子有两个路径段,hammers和index.html。hammers路径段有参数sale，其值为false。index.html段有参数graphics,其值为true。

## 查询字符串
```bash
http://www.joes-hardware.com/inventory-check.cgi?item=12731
```
字符?右边的内容被称为查询组件，查询字符串很多也是以键值对的形式出现，键值对之间用"&"隔开。

## 片段
对一些带有章节的大型文本文档文件，为了引用部分资源或资源的一个片段，URL支持使用片段(frag)组件来表示一个资源内部的片段。比如，URL可以指向HTML文档的一个特定图片或小节,最前面有个字符"#"。比如：
```bash
http://www.joes-hardware.com/tools.html#drills
```

## 字符限制
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020061811303988.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)


