### 目录
- [语言](#语言)
    - [Golang](#Golang)
	 	- [核心知识](#Golang核心知识)
	 	- [源码知识](#Golang源码知识)
	 	- [并发编程](#并发编程)
	- [Golang开源相关](#Golang开源相关)
	    - [Etcd分布式键值存储](#Etcd分布式键值存储)
	    - [Groupcache缓存](#Groupcache缓存)
	- [设计模式](#设计模式)
    - [数据结构](#数据结构)
- [算法](#算法)
- [数据库](#数据库)
    - [mysql](#mysql)
- [建模](#建模)
- [架构设计核心](#架构设计核心)
    - [高并发](#高并发)
- [分布式](#分布式)
- [微服务](#微服务)
- [容器编排](#容器编排)
- [计算机原理](#计算机原理)
- [藏经阁](#藏经阁)
    - [中间件](#中间件)
        - [redis](#redis)
           - [核心知识](#redis核心知识)
           - [源码知识](#redis源码知识)
- [大数据](#大数据)
    - [Spark](#Spark)


# 语言
## Golang
### Golang核心知识
- [Map入门](https://github.com/friendlyhank/toBeTopgopher/blob/master/golang/base/map.md)
- [go module初探](https://github.com/friendlyhank/toBeTopgopher/blob/master/golang/base/go_module初探.md)

### Golang源码知识
- [map解读](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)
- [pgm解读](https://github.com/friendlyhank/go-source/blob/master/runtime/golang%20pgm.md)
- [双链表](https://github.com/friendlyhank/go-source/blob/master/container/list/list.md)
- [channel讲解](https://github.com/friendlyhank/toBeTopgopher/blob/master/golang/source/go_channel.md)
- [select讲解](https://github.com/friendlyhank/toBeTopgopher/blob/master/golang/source/golang_select.md)
- [go1.13.5版本源码注释](https://github.com/friendlyhank/go1.13.5-annotated)

### 并发编程

## Golang开源相关

### Etcd分布式键值存储
### Groupcache缓存

## 设计模式

# 数据结构
- [数组]()
- [切片]()
- [map解读](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)
- [redis6.0源码双端链表](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis6.0源码双端链表.md)
- [环形双链表](https://github.com/friendlyhank/go-source/blob/master/container/list/list.md)

# 算法

# 数据库
## mysql
- [基本数据类型](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/mysql数据类型.md)
- [join连接查询](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/mysql_join.md)
- [SQL优化连弹一之慢查询日志开启](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/SQL优化连弹一之慢查询日志开启.md)
- [SQL优化连弹二之show status](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/SQL优化连弹二之show_status.md)
- [SQL优化连弹三之explain](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/SQL优化连弹三之explain.md)
- [SQL优化连弹四之show profile](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/SQL优化连弹四之show_profile.md)
- [分布式事务XA](https://github.com/friendlyhank/toBeTopgopher/blob/master/mysql/mysql分布式事务XA.md)
# 建模

# 架构设计核心
## 高并发

### 应用级缓存
- [缓存使用模式](https://juejin.im/post/5af5b2c36fb9a07ac65318bd)


### 多级缓存
- [深入理解分布式之数据库和缓存双写一致性方案解析](https://juejin.im/post/5b923de95188255c8e725c18)

# 分布式
- [一致Hash算法](https://github.com/friendlyhank/toBeTopgopher/blob/master/distributed_system/hash一致性算法讲解.md)

# 微服务

# 容器编排

## Kubernetes

# 计算机原理

# 藏经阁

## 中间件

### redis

#### redis核心知识
- [redis分布式锁官方解读](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis分布式锁官方解读.md) 💪💪
- [持久化RDB和AOF](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis持久化RDB和AOF.md)
- [主从复制之配置](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis主从复制之配置.md)
- [主从复制之原理](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis主从复制的原理.md)
- [sentinel概述和配置安装](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/sentinel概述和配置安装.md)
- [sentinel api](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/sentinel_api.md)
- [sentinel客户端实现](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/sentinel客户端实现.md)
- [sentinel实现原理](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/sentinel实现原理.md)
- [redis集群](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis集群.md)

#### redis源码知识
- [redis6.0源码动态字符串SDS](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis6.0源码动态字符串SDS.md)
- [redis6.0源码双端链表](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis6.0源码双端链表.md)
- [redis6.0源码字典的实现](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis6.0源码字典的实现.md)
- [redis6.0源码跳跃表](https://github.com/friendlyhank/toBeTopgopher/blob/master/redis/redis6.0源码跳跃表.md)

# 大数据

### Spark
