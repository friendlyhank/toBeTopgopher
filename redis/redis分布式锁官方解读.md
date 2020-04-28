## 概述
在许多环境中，不同的进程必须以互斥的方式使用共享资源进行操作，就用到分布式锁。

**分布式锁具有的特性：**
 - **安全性**：在任意给定时刻,只有一个客户端可以获得锁,具有排它性。
 - **容错性A**：避免死锁。客户端最终一定可以获得锁，即使锁住某个资源的客户端在释放锁之前崩溃或者发生网络分区。
 - **容错性B**：只要大多数的节点都处于运行状态，客户端就可以获取和释放锁。

## 单个实例实现的锁
让我们看看在单例中是如何实现锁,命令如下：
```go
SET resource_name my_random_value NX PX 30000
```
SET NX表示在key值不存在的情况给key赋值，PX表示为键设置毫秒级过期时间。键值为随机值，键值必须唯一。

设置唯一随机值是为了保证释放锁的安全性，当这个key对应的value一致的时候才可以删除这个key，这里可以通过Lua脚本完成：
```go
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```
比如A客户端在获得锁之后，可能因为某些操作阻塞超过了锁的过期时间，这时候B客户端获得了锁,然后A客户端的操作恢复之后又把B客户端的锁删除了，所以这里要保证释放锁的安全性。

通过上面的两个操作，我们可以完成获得锁和释放锁操作。如果这个系统不宕机，那么单点的锁服务已经足够安全,接下来我们开始把场景扩展到具备分布式的容错性更好的实现方法。

## Redlock算法介绍
在算法的分布式版本中,假设我们有N个master节点,这些节点是完全独立的,无需设置主从复制,这里假定N的数量为5作为示例。
 1. 以毫秒为单位获取当前时间
 2. 按照顺序让N个redis实例设置相同的键名和随机值来获取锁定。在每个实例获取锁的过程,要为每一个客户端设置一个超时时间。例如，如果锁的有效时间为10秒，则每个key的超时时间设置为5-50毫秒之间，这样可以防止客户端长时间与处于故障状态的Redis节点通信时保持阻塞，如果一个实例不可用，应该尽快尝试下一个实例。
 3. 设置完成之后通过获取当前时间减去步骤1的时间，来计算获取锁所花费的时间。当且仅当客户端能够在大多数实例(至少3个)中获取锁，并且获取锁所花费的时间小于锁的有效时间，则认为客户端获得了锁。
 4.如果客户端未获得锁(获取锁的实例少于N/2+1或者获取锁花费时间超过锁的有效时间)，则要尝试释放所有实例的锁。

## 设计要点
设计要点会根据官方推荐的go版实现的分布式锁源码[redsync](https://github.com/go-redsync/redsync)来进行分析讲解，里面也包含Redlock算法的实现。

首先来看看具体的两个结构体：
[Redsync](https://github.com/go-redsync/redsync/blob/master/redsync.go)定义了多个redis连接池，用于多个节点的获取和释放锁的操作。
```
type Redsync struct {
	pools []Pool
}
```

[Mutex](https://github.com/go-redsync/redsync/blob/master/mutex.go)是分布式锁的定义
```
type Mutex struct {
	name   string  				//名称
	expiry time.Duration    //锁的有效时间

	tries     int   //尝试次数
	delayFunc DelayFunc //失败尝试设置延迟

	factor float64 //误差系数控制

	quorum int //投票数 一般为节点数/2+1，节点数为奇数

	genValueFunc func() (string, error)  //加密函数，生成唯一随机串
	value        string  //值，默认就是唯一随机串
	until        time.Time //过期时间

	pools []Pool //连接池
}
```
在[Redsync](https://github.com/go-redsync/redsync/blob/master/redsync.go)中有专门得方法NewMutex对分布式锁得参数进行初始化，这里不做详细讲解,

### 获取锁
```
func (m *Mutex) Lock() error {
	value, err := m.genValueFunc() //生成唯一随机串，默认用base64
	if err != nil {
		return err
	}
	//失败尝试
	for i := 0; i < m.tries; i++ {
		if i != 0 {
			time.Sleep(m.delayFunc(i))
		}

		start := time.Now()
	   //尝试异步去获取锁
		n, err := m.actOnPoolsAsync(func(pool Pool) (bool, error) {
			return m.acquire(pool, value)
		})
		if n == 0 && err != nil {
			return err
		}

		now := time.Now()
		//过期时间 = 有效时间值-获取锁消耗的时间值-有效时间值*误差系数
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.factor)))
		//成功节点数>= 节点数/2+1 && 未过期
		if n >= m.quorum && now.Before(until) {
			m.value = value
			m.until = until
			return nil
		}
		//获取锁失败，异步释放锁
		m.actOnPoolsAsync(func(pool Pool) (bool, error) {
			return m.release(pool, value)
		})
	}

	return ErrFailed
}
```
这里获得锁得过程实际上就是对Redlock算法得实现：
**失败重试：** 当客户端无法获取锁得时候会设置一个随机值重试，这个随机值的重试时间应当和当次申请锁的时间错开，减少脑裂的可能。

一个客户端在所有redis实例中申请的时间越短，发生脑裂的时间窗口越小，所以要用非阻塞的方式，这里[actOnPoolsAsync](https://github.com/go-redsync/redsync/blob/master/mutex.go)同时向多个redis实例发送异步set请求。
```go
func (m *Mutex) actOnPoolsAsync(actFn func(Pool) (bool, error)) (int, error) {
	type result struct {
		Status bool
		Err    error
	}

	ch := make(chan result)
	for _, pool := range m.pools {
		go func(pool Pool) {
			r := result{}
			r.Status, r.Err = actFn(pool)
			ch <- r
		}(pool)
	}
	n := 0
	var err error
	for range m.pools {
		r := <-ch
		if r.Status {
			n++
		} else if r.Err != nil {
			err = multierror.Append(err, r.Err)
		}
	}
	return n, err
}
```
方法中actFn就是调用[acquire](https://github.com/go-redsync/redsync/blob/master/mutex.go)去设置锁：
```go
func (m *Mutex) acquire(pool Pool, value string) (bool, error) {
	conn := pool.Get()
	defer conn.Close()
	reply, err := redis.String(conn.Do("SET", m.name, value, "NX", "PX", int(m.expiry/time.Millisecond)))
	if err != nil {
		if err == redis.ErrNil {
			return false, nil
		}
		return false, err
	}
	return reply == "OK", nil
}
```

### 释放锁
```go
func (m *Mutex) Unlock() (bool, error) {
	n, err := m.actOnPoolsAsync(func(pool Pool) (bool, error) {
		return m.release(pool, m.value)
	})
	if n < m.quorum {
		return false, err
	}
	return true, nil
}
```

## 常见问题
为什么不能用主从复制实现故障转移？
答：因为Redis的复制是异步的
	 此模型存在明显的竞态条件,会导致多个客户端同时持有一个锁：
 1. 客户端A获取主服务器的锁
 2. 在master将锁信息备份到slave节点之前,主节点宕机
 3. salve节点晋升为master节点
 4. 客户端B从新master获得锁

[官方参考](https://redis.io/topics/distlock)
[更多redis系列文章,欢迎关注我的github](https://github.com/friendlyhank/toBeTopgopher#redis)
