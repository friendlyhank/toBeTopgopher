> redis的字典使用哈希表作为底层实现。
## 字典操作
### 数据结构定义
结构体定义在dict.h中。
字典的结构体定义：
```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
ht是一个长度为2的数组，对应的是两个哈希表，一般使用使用ht[0],ht[1]主要在扩容和缩容时使用。

哈希表结构体定义：
```c
typedef struct dictht {
	//哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;
   	//哈希表大小的掩码，用于计算索引值
   	//总是等于size-1
    unsigned long sizemask;
   	//已有节点的数量
    unsigned long used;
} dictht;
```
table是一个数组，对应的是多个哈希表节点dictEntry

哈希表节点key/value结构体定义：
```c
typedef struct dictEntry {
	//键
    void *key;
    //值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    //指向下一个哈希表节点,形成链表
    struct dictEntry *next;
} dictEntry;
```
哈希表每个节点都保存着一个键值对，key就是键值对的键，v属性就是对应键值对的值，v可以是一个指针也可以是uint64_t,整数也可以是int64_t整数。
next是一个链表，指向着下一个哈希表节点，这个指针可以将多个哈希值相同的键值对连接在一起，以此来解决哈希冲突问题。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200513103428416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
### 哈希算法
redis字典实现是使用链地址法，哈希算法具体方式为：

 1. 计算键key的hash值
	```c
	hash = (ht)->type->hashFunction(key)
	```
 
 2. 通过hash与sizemask的位运算计算出哈希table数组(有些语言也叫桶)对应的索引
	```c
	h = dictHashKey(ht, key) & ht->sizemask;
	```
 3. 插入的时候，可能会出现键被分配到同一个哈希表数组的索引上，引发哈希冲突，解决哈希冲突的方式是每次把新增节点往单向链表的头部插入，每个节点会记录下一个节点的信息next。
 4. 每次插入的时候，会检查是否需要是否需要扩容，扩容由哈希因子决定的,负载因子=哈希表已保存的节点数/哈希表大小
	```c
	d->ht[0].used/d->ht[0].size
	```
 5. 扩容和收缩通过rehash完成,扩容或收缩表需要把ht[0]的所有键rehash到ht[1]里面，考虑服务器性能原因rehash并不会一次性地把所有ht[0]所有键rehash到ht[1]里，而是渐进式、分多次的完成。

### 字典的创建
```c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{	
	//分配创建字典的内存
    dict *d = zmalloc(sizeof(*d));
	//对字典进行初始化
    _dictInit(d,type,privDataPtr);
    return d;
}


int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
	//初始化hash表
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```

### 渐近式rehash
渐进式rehash方法在dict.c/dictRehash中
```c
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```

```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;
        //防止rehashidx越界
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
       
        while(de) {
            uint64_t h;

            nextde = de->next;
          
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    return 1;
}
```
 1. 从ht[0]根据rehashidx遍历,获得ht[0]表中的哈希节点信息
 2. 每次遍历出哈希节点信息就把它迁移到ht[1]
 3. 如果ht[0]中没有hash节点，把ht[1]设置称ht[0]
 4. 重置ht[1]

扩展和缩容表实际上不是一次完成的，因为考虑键值对可能非常大，如果一次完成可能会非常消耗性能。

因为rehash是渐进式的，所以也会引发问题是在删除、查找、更新等操作的时候都在两个哈希表中进行。
另外在rehash期间，新添加的字典的键值对会保存到ht[1]里面，而ht[0]则不会进行任何添加操作，保证ht[0]只减不增，随着rehash操作执行最终变为空表。

### 添加元素到字典
添加的方法在dict.c/dictAdd
```c
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key,NULL);

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);
    return DICT_OK;
}
```
dictAdd会调用dictAddRaw生成一个新的哈希节点，然后调用dictSetVal去设置这个哈希节点的val值,设置哈希节点的key值是在dictAddRaw方法里。
```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;
	//如果正在rehash，进行渐进rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;
    
    dictSetKey(d, entry, key);
    return entry;
}
```
 1. 获取哈希表对应的索引index
 2. 根据是否rehash状态获取ht[1]还是获取ht[0]，如果正在refash,则用哈希表ht[1]进行插入
 3. 分配内存生成新的hash节点，然后在链表头部插入这个hash节点
 4. 更新对应hash节点信息

查找table的索引方法是在dict.c/_dictKeyIndex方法
```c
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
	...
	 for (table = 0; table <= 1; table++) {
		idx = hash & d->ht[table].sizemask;
		//获取到哈希表节点的链表
		 he = d->ht[table].table[idx];
		  while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
	}
	...
	return idx;
}
```

 1. 分别从字典得ht[0]、ht[1]中查找
 2. 计算哈希表dictht中table的索引,计算方式为：idx = hash & sizemask,sizemask总会比size小1。
 3. 根据table的索引得到hash表节点，然后去遍历链表。 
 4. 如果key不存在，返回table的索引，如果存在返回-1,并把节点信息通过existing返回。

### 字典元素替换
```c
int dictReplace(dict *d, void *key, void *val)
{
    dictEntry *entry, *existing, auxentry;
    
    entry = dictAddRaw(d,key,&existing);
    if (entry) {
        dictSetVal(d, entry, val);
        return 1;
    }

    auxentry = *existing;
    dictSetVal(d, existing, val);
    dictFreeVal(d, &auxentry);
    return 0;
}
```
 1. 调用dictAddRaw添加hash节点，如果节点已经存在则entry会返回null,并把已存在节点返回existing
 2. 如果不存在，则直接添加，并返回1
 3. 如果不存在，修改节点的值


### 查找给定键的值
```c
void *dictFetchValue(dict *d, const void *key) {
    dictEntry *he;

    he = dictFind(d,key);
    return he ? dictGetVal(he) : NULL;
}
```
```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (d->ht[0].used + d->ht[1].used == 0) return NULL; /* dict is empty */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

 1. 计算指定key的hash值
 2. 遍历hash表，根据hash值和sizemask计算出table的索引
 3. 取得hash节点链表信息进行遍历
 4. 找到节点信息后返回

## 迭代器
###  用字典生成迭代器
```c
dictIterator *dictGetIterator(dict *d)
{
    dictIterator *iter = zmalloc(sizeof(*iter));

    iter->d = d;
    iter->table = 0;
    iter->index = -1;
    iter->safe = 0;
    iter->entry = NULL;
    iter->nextEntry = NULL;
    return iter;
}
```

## 迭代器遍历字典
实际上遍历字典，因为最底层字典是链表存储的，所以遍历字典很简单相当于遍历字典就可以了,只是因为收缩和扩容的原因，需要遍历2个hash表。
```c
dictEntry *dictNext(dictIterator *iter)
{
    while (1) {
        if (iter->entry == NULL) {
            dictht *ht = &iter->d->ht[iter->table];
            if (iter->index == -1 && iter->table == 0) {
                if (iter->safe)
                    iter->d->iterators++;
                else
                    iter->fingerprint = dictFingerprint(iter->d);
            }
            iter->index++;
            if (iter->index >= (long) ht->size) {
                if (dictIsRehashing(iter->d) && iter->table == 0) {
                    iter->table++;
                    iter->index = 0;
                    ht = &iter->d->ht[1];
                } else {
                    break;
                }
            }
            iter->entry = ht->table[iter->index];
        } else {
            iter->entry = iter->nextEntry;
        }
        if (iter->entry) {
            /* We need to save the 'next' here, the iterator user
             * may delete the entry we are returning. */
            iter->nextEntry = iter->entry->next;
            return iter->entry;
        }
    }
    return NULL;
}
```

 1. 如果字典正在rehash,说明正在使用一号表，设置节点列表为一号表ht[1]
 2. 遍历链表直接返回hash节点信息

更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)
