## github下载etcd
```
$ git clone https://github.com/etcd-io/etcd.git
$ cd etcd
$chmod +x build
$ ./build
```

## 运行服务端
```
cd /bin
./etcd
```

## 客户端
```
cd /bin
./etcdctl put foo bar
./etcdctl get foo
```

[安装参考](https://github.com/etcd-io/etcd/blob/master/Documentation/dl_build.md)

