## 结构体
哈希表结构体：
```go
// A header for a Go map.
type hmap struct {
	count     int   //表示当前哈希表中元素的数量
	flags     uint8  //标记
	B         uint8  // 桶的数量2^B
	noverflow uint16 // 溢出桶数量统计
	hash0     uint32 // 哈希因子,用于计算出hash值

	buckets    unsafe.Pointer //2^B的桶
	oldbuckets unsafe.Pointer //扩容时用于保存之前的buckets
	nevacuate  uintptr//扩容之后数据迁移的计数器,记录下次迁移的位置   

	extra *mapextra // 额外的map字段,存储溢出桶信息
}
```
 - count表示当前哈希表中元素的数量
 - flags 表示哈希表的标记 1表示buckets正在被使用 2表示oldbuckets正在被使用 4表示哈希正在被写入 8表示哈希是等量扩容
 - B表示2的B次方系数表示,如B是5,那么桶的数量就有2^5个

哈希表结构体溢出桶信息：
```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	overflow    *[]*bmap //所有的溢出桶数据
	oldoverflow *[]*bmap //所有旧溢出桶数据

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap //指向下一个溢出桶
}
```

桶结构体：
```go
// A bucket for a Go map.
type bmap struct {
	//tophash用于快速从桶中找到对应键值对的位置
	tophash [bucketCnt]uint8
}
```

## makemap初始化
哈希表的初始化如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032210013721.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
当我们调用make关键字对map进行初始化的时候,Go语言编译器会将它转化成makemap。
```go
make(map[k]v,hint)//hint表示元素的个数
```

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	//计算内存空间和判断是否内存溢出
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	//计算出指数B,那么桶的数量表示2^B
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	if h.B != 0 {
		var nextOverflow *bmap
		//根据B去创建对应的桶和溢出桶
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```
主要的步骤为:
 1. 计算出需要的内存空间并且判断内存是否溢出
 2. hmap没有的情况进行初始化，并设置hash0表示hash因子
 3. 计算出指数B,桶的数量表示为2^B,通过makeBucketArray去创建对应的桶和溢出桶

我们来看看makeBucketArray是如何创建桶和溢出桶
```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	//当指数B大于等于4,增加额外的溢出桶
	if b >= 4 {
		//溢出桶个数为2^(B-4)
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
		//生成对应数量的桶
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		//得到对应的溢出桶
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
 - 当桶的数量小于2^4时，可能数据较少，使用溢出桶的可能较低，所以不会创建溢出桶
 - 当桶的数量大于2^4时, 可能数据较多，使用溢出桶可能性较大,所以会创建额外的
 2^B-4的桶。

## 写入
写入过程示意图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210322095353710.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {

	.....
	//计算出hash值
	hash :=t.hasher(key,uintptr(h.hash0))
	
	//更新状态为正在写入
	h.flags ^= hashWriting
	
again:
	//通过hash获取对应的桶
	bucket := hash & bucketMask(h.B)
	b :=(*bmap)(unsafe.Pointer(uintptr(h.buckets)+bucket*uintptr(t.bucketsize)))
	//计算出tophash
	top :=tophash(hash)
	
	var inserti *uint8//记录插入的tophash
	var insertk unsafe.Pointer//记录插入的key值地址
	var elem unsafe.Pointer//记录插入的value值地址
	
bucketloop:
	for{
		for i :=uintptr(0);i < bucketCnt;i++{
			//判断tophash是否相等
			if b.tophash[i] != top {
				//如果tophash不相等并且等于空,则可以插入该位置
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					//获取对应插入key和value的指针地址
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			
			//走到这里,说明已经存在,获得指定的key和value在桶得位置地址
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			//如果是指针，则要转化为指针
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			//判断key值是否相等
			if !t.key.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			//如果key值需要修改，那么修改key值
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			//获取value元素地址
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
			
			//未找到可插入的位置,找一下有没溢出桶，如果有继续执行写入操作
			ovf := b.overflow(t)
			if ovf == nil{
				break
			}
			b = ovf
		}
	}
	
	if inserti == nil {
		//如果在正常桶和溢出桶中都未找到插入的位置，那么得到一个新的溢出桶执行插入
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}
	
	.....
	//将key值信息插入桶中指定位置
	typedmemmove(t.key, insertk, key)
	*inserti = top//更新tophash值
	h.count++
	
done:
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem //返回value的指针地址
}
```
 1. 计算key的hash值,通过hash的高八位和低B为分别确定tophash和桶的序号
  **tophash是什么?**
tophash是用来快速定位key和value的位置的,在查找或删除过程如果高8位hash都不相等，那么就没必要再去比较key值是否相等了，效率相对会高一些。
  **如何定位到哪个桶执行插入?**
例如哈希表对应2^4个桶,即B是4,某个key的hash二进制值是如下值，那么如图可知该key对应的tophash值为10001100,即140，桶的值为0111,即是桶的序号为7。
 ```go
hash := 100011001101111001110010010110000001111010110000100101011010111
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032211403258.png)
确定tophash和桶的序号之后就可以进行插入操作。

 2. 每个桶可以存储8个tophash、8个key、8个value,遍历桶中的tophash,如果tophash不相等且是空的,说明该位置可以插入，分别获取对应位置key和value的地址并更新tophash。

 **桶的结构到底是怎样的？**
桶的结构体并不是上面提到的tophash [8]uint8,因为go是不支持泛型的，所以在编译过程中才会根据具体的类型确定,实际上桶的结构可以表示为：
```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210322122353220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
插入到具体某个桶的示意图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210322195225778.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

 3. 如果遍历桶中的tophash是相等的,说明执行过插入操作,同样的获取对应位置的key和value的地址。
 4. 如果当前桶找不到插入的位置，说明可能满了，尝试去溢出桶中执行插入操作。
 5. 如果未找到溢出桶或溢出桶也满了，那么调用newoverflow获取一个溢出桶执行插入操作，然后同样是获取溢出桶插入位置的key和value地址并更新tophash。

一开始的时候正常桶和溢出桶是一块连续的内存空间，后来因为某个桶满了需要用到溢出桶执行插入操作,那么这时候这个溢出桶就会被链接到这个桶的最后方，形成链表,这个方法主要通过newoverflow来完成的。
```go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
	var ovf *bmap
	if h.extra != nil && h.extra.nextOverflow != nil{
		//从预分配好的溢出桶中获取一个桶
		ovf = h.extra.nextOverflow
		//如果下一个溢出桶不为空,还记得makeBucketArray方法里，最后一个溢出桶尾部链接的是buckets首地址,当不为nil的时候才是表示用完了
		if ovf.overflow(t) == nil{
			h.extra.nextOverflow =(*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
		}else{
			//如果溢出桶用完了,标记extra.nextOverflow为nil
			ovf.setoverflow(t,nil)
			h.extra.nextOverflow = nil
		}
	}else{
		//溢出桶被用完了,从内存中分配一个溢出桶
		ovf =(*bmap)(newobject(t.bucket))
	}
	h.incrnoverflow()//溢出统计,扩容时也会参考这个
	//如果不是指针
	if t.bucket.ptrdata == 0{
		h.createOverflow()
		*h.extra.overflow = append(*h.extra.overflow, ovf)//将溢出桶加入到h.extra.overflow
	}
	b.setoverflow(t,ovf) //将要用的溢出桶链接在当前桶的最尾部
	return ovf
}
```

这里还有一个需要注意的点，插入过程最后elem(value)返回的仅仅是指针地址，真正的数据写入其实是在编译的时候完成的。
```go
0x00e0 00224 (main.go:5)        CALL    runtime.mapassign(SB)           //写入操作
0x00e5 00229 (main.go:5)        MOVQ    24(SP), DI                      //DI = &value
0x0104 00260 (main.go:5)        LEAQ    go.string."hello world"(SB), AX //AX =&"hello world"
0x010b 00267 (main.go:5)        MOVQ    AX, (DI)                        //*DI = AX
```
## 扩容
其实哈希表还没有我们想象的这么简单，因为在写入过程中还涉及到扩容的操作，扩容之后还涉及到数据迁移的过程，还需要把旧桶的数据迁移到新的桶之中,先来看看扩容的过程。
```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	.....
	//判断是否扩容的条件
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	.....
}
```
判断是否扩容的条件：

 1. 哈希表不是正在扩容的状态
 2. 元素的数量 > 2^B次方(桶的数量) * 6.5,6.5表示为装载因子,很容易理解装载因子最大为8(一个桶能装载的元素数量)
 3. 溢出桶过多,当前已经使用的溢出桶数量 >=2^B次方(桶的数量) ,B最大为15

上述条件满足就会触发扩容机制,扩容分为两种，一种是等量扩容和2倍扩容：
```go
func hashGrow(t *maptype,h *hmap){
	//没有超出装载因子是等量扩容
	bigger := uint8(1)
	if !overLoadFactor(h.count+1,h.B){ 
		bigger = 0
		h.flags |= sameSizeGrow
	}
	
	//将当前桶设置为旧桶,调用makeBucketArray分配新的top和溢出桶
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	//更新哈希的标志
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0{
		flags |= oldIterator
	}
	//更新哈希表的信息
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0
	
	//同理将已经使用的溢出桶设置为旧的溢出桶
	if h.extra != nil && h.extra.overflow != nil{
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow =  nil
	}
	//更新溢出桶相关信息
	if nextOverflow != nil{
		if h.extra == nil{
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}
}
```

 1. 调用overLoadFactor函数判断装载因子是否超出，如果超出则进行2倍扩容，如果不是则进行等量扩容。
 2. 如果是等量扩容,只需要分配原来2^B数量的桶即可，如果是2倍扩容，只需B+1，即是2B+1数量的桶。
 3. 同样的调用makeBucketArray函数来分配对应数量的桶，这时候h.buckets会变为h.oldbuckets,新创建的桶newbuckets会被设置为h.buckets。
 4. 和桶的设置一样，当前正在使用的h.extra.overflow溢出桶会变为h.extra.oldoverflow,h.extra.overflow会被设置成nil，同时h.extra.nextOverflow也会更新指向到新创建的未使用的溢出桶。
## 迁移
在上面扩容hashGrow方法中仅仅是完成扩容的操作，并没有进行数据的迁移，实际上在扩容完成后，在下次写入操作或删除操作之前，就会进行数据的迁移。
```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	.....
	//如果正在扩容,进行数据迁移
	if h.growing(){
		growWork(t, h, bucket)
	}
	.....
}
```

迁移的话会以桶为单位,先对当前要操作的桶进行数据的迁移，同时为了加快进度，也会从上次标记的迁移位置继续执行一次迁移的操作，真正的数据迁移调用的方法是evacuate。
```go
func growWork(t *maptype,h *hmap,bucket uintptr){
	//对当前要使用的桶进行迁移
	evacuate(t,h,bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing(){
		//从迁移的标志位置继续迁移
		evacuate(t,h,h.nevacuate)
	}
}
```

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	//获得要迁移的旧桶
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	//旧桶的数量
	newbit := h.noldbuckets()
	//桶里有数据则需要数据迁移
	if !evacuated(b) {
		//定义一个大小为2的evacDst,x分别映射新桶的低位,y映射新桶的高位,这个会详细解释
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))
		//如果是2倍扩容,则要映射高位,等量扩容则不需要一一对应
		if !h.sameSizeGrow() {
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}
		
		//遍历旧桶
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				//如果是2倍扩容,通过hash计算是插入低位还是高位
				if !h.sameSizeGrow() {
					hash := t.hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination
				
				//计算出新桶的低位或高位插入之后，将数据进行迁移,正常桶转移完转移溢出桶的数据
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)//获取一个溢出桶
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				
				//保存tophash，并复制key和value的数据
				dst.b.tophash[dst.i&(bucketCnt-1)] = top 
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				dst.i++
				
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		//释放旧桶的数据
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}
	//标记转移的位置，方便下次迁移
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```
 1. 获取指定要迁移的旧桶，定义一个[2]evacDst,如果是等量扩容,旧桶和新桶的插入的序号是一样的。如果是2倍扩容，旧桶迁移到新桶可能会对应插入的低位和高位,用evacDst分别映射低位和高位的桶地址，做好迁移的准备。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210323112211490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
**2倍扩容之后桶数据的迁移为什么会对应低位和高位两个位置？**
上面提到,桶得序号是由hash的低B位计算出来的，当桶发生扩容之后B指数比原来加一，经过计算桶的序号可能发生改变，这个取决于第低B位的值是0还是1，例如假设未扩容前有2的4次方个桶，计算hash的低4为0111,即桶的序号为7,这时候发生扩容，桶的数量变为2的5次方个桶,这时候桶的序号变为取hash的低5位，即10111,桶的序号变为7+16(旧桶的个数)，即是23号桶,另外一种，假设新的桶的低B位还是00111,那么迁移到新桶的序号就是和旧桶是一样的，还是序号7。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210323114333102.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 2. 遍历旧桶的数据，获取key的hash并计算插入映射到新桶的evacDst低位还是高位,然后进行迁移操作，更新tophash并复制key和value的值。
 3. 如果溢出桶有数据也会遍历溢出桶数据进行迁移操作，原理同上。
 4. 桶的数据迁移完后旧桶的将会被释放并对数据迁移的位置做上标记，下次迁移方便从标记继续执行迁移。

如果你熟悉了上面的写入、扩容、迁移的过程，后面的写入就会非常容易理解。
## 查找
go的哈希查找有两种方式,一种是不返回ok的对应的源码方法为runtime.mapaccess1,另外返回ok的函数对应源码方法为runtime.mapaccess2,两种实现基本相同,下面以runtime.mapaccess2为例。
```go
v,_ := map[key]
v,ok := map[key]
```

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
    .....
	//得到hash并计算出桶得序号
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	
	//如果旧桶有数据且未发生数据迁移，那么切换到旧桶里去找数据
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	//遍历桶
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			//如果tophash不相等并且桶内没有元素,那么跳出循环不再遍历,说明没有找到想要的元素
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			
			//tophash相等,找到对应的元素
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			//判断key值是否相等,如果相等,则返回value值
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e, true
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0]), false
}
```
## 删除
```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	.....
	hash := t.hasher(key, uintptr(h.hash0))

	bucket := hash & bucketMask(h.B)
	
	//如果正在扩容,那么对数据进行迁移
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	
	//遍历桶
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			//如果tophash不相等并且桶内没有元素,那么跳出循环不再遍历,说明没有找到想要的元素
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			//tophash相等,找到对应的元素
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.key.equal(key, k2) {
				continue
			}
			//对key值的空间清空并释放
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			//对value值的空间清空并释放
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
			//设置tophash为空
			b.tophash[i] = emptyOne
		notLast:
			h.count-- //元素个数减一
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```

**特定类型的数据优化**
在哈希表的读取、写入、删除、数据迁移的时候，会针对特定或常用的数据类型进行优化，比如uint64,uint32,string类型等，因为是特定的数据类型，所以可以节省很多操作，优化效率，有兴趣的同学可以自己找来看看，这里不再做讲解。
| key类型 | 源码文件 |
|--|--|
| uint64 | runtime/map_fast64.go |
| uint32| runtime/map_fast32.go |
| string| runtime/map_faststr.go |

