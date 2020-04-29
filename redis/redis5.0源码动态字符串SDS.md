redis中没有使用c语言的字符串，而是自己构建了SDS(simple dynamic string),解析为简单动态字符串。c语言字符串表示方式不能满足redis对字符串在安全性、效率以及功能方面的要求。~~动态字符串在哪方面使用。~~ 

**sds与c字符串的异同：**
| C字符串 |SDS  |
|--|--|
|获取字符串长度的复杂度为O(N)  | 获取字符串长度的复杂度为O(1) |
|API是不安全的，可能会造成缓冲区溢出(手动分配)  | API是安全的，不会造成缓冲区溢出(预分配)|
|修改字符串长度N次必然需要执行N次内存重分配  | 修改字符串长度N次最多需要N次内存重分配(预分配和惰性空间释放)|
|只能保存文本数据 | 可以保存文本或者二进制数据|
|可以使用所有<string.h>库中的函数 | 可以使用一部分<string.h>库中的函数|

## 动态字符串
redis的sds结构实现字符串的基础。具体的文件有两个sds.c sds.h
**sds定义**
```c
typedef char *sds;
```
sds字符串根据字符串的长度，划分了五种结构体sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64,分别对应的类型为SDS_TYPE_5、SDS_TYPE_8、SDS_TYPE_16、SDS_TYPE_32、SDS_TYPE_64

每个sds 所能存取的最大字符串长度为：
sdshdr5最大为32(2^5)
sdshdr8最大为0xff(2^8-1)
sdshdr16最大为0xffff(2^16-1)
sdshdr32最大为0xffffffff(2^32-1)
sdshdr64最大为(2^64-1)

**sds结构体**
sds每个类型的结构体大体相同,除了sdshdr5是没有len、alloc的，其他的结构体就只是len和alloc定义数据类型上的区别。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200429083544766.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 - 当字符串长度为0的时候，通常被认为要进行append操作，所以sds类型为SDS_TYPE_8
 - 比如SDS_TYPE_8长度是2^8，因为sds和c字符串一样末尾有空格'\0'占位，区间范围左侧又是闭区间,所以区间范围为[32,255)

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
 - __attribute__ ((__packed__))告诉编译分配的是紧凑内存，而不是字节对齐的方式。
 - len表示字符串已使用的长度
 - alloc表示字符串的容量
 - flags表示字符串类型标记SDS_TYPE_5、SDS_TYPE_8、SDS_TYPE_16、SDS_TYPE_32、SDS_TYPE_64
 - buf[]表示柔性数组。在分配内存的时候会指向字符串的内容

## 创建sds
```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    //根据大小获取SDS的类型
    char type = sdsReqType(initlen);
    
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; 
	//分配内存
    sh = s_malloc(hdrlen+initlen+1);//sds和字符串一样，尾部有空格符所以要+1
    if (sh == NULL) return NULL;
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    //根据类型初始化长度、容量、标记
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);//复制数据
    s[initlen] = '\0';
    return s;
}
```
 1. 根据初始大小设置长度

## 空间预分配
```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
	//空间充足，无需再分配空间
	if (avail >= addlen) return s;
	
	//计算出新增sds需要空间长度
	newlen = (len+addlen);
	
	//如果新增sds需要空间长度<SDS_MAX_PREALLOC(大小为1M)
	//预分配新增sds需要空间长度的2倍
	//否则预分配新增sds需要空间长度+SDS_MAX_PREALLOC
	if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
        
	//重新获取sds类型
	type = sdsReqType(newlen);
	hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
    	//如果sds类型不变，对原来的sds空间进行重新分配
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) {
            s_free(sh);
            return NULL;
        }
        s = (char*)newsh+hdrlen;
    } else {
       //分配新的空间
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
}
```

SDS_MAX_PREALLOC的值为1024*1024

## 惰性空间释放
