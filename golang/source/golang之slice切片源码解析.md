golang的切片实现是在包runtime/slice.go
## 定义
切片定义包含array指向数组的指针，是一块连续的内存空间,len代表切片的长度,cap代表切片的容量,cap总是大于等于len。
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210129150810334.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
## make初始化
先来看看slice的初始化,slice的初始化可以通过make关键字,传入type、len、cap。

> make(Type,len,cap)

```go
var numbers = make([]int64,5,6)
```

slice的make初始化主要通过runtime.makeslice来完成,先计算出需要的内存空间大小，然后再分配内存。
```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	//计算需要分配的内存空间和内存是否有溢出
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}
	//分配内存
	return mallocgc(mem, et, true)
}
```
内存空间大小的计算公式为：
> 内存空间大小 = 切片中元素大小 * 容量大小

## append扩容
当使用append的时候就可能会触发切片的扩容机制,扩容是调用runtime.growslice,具体步骤为：
```go
func growslice(et *_type, old slice, cap int) slice {
	.....
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
}
.....
```
 1. 在一开始旧切片old.cap可能还未初始化,old.cap为0,这时候new.cap直接等于预期的容量cap
 2. 当旧切片的长度old.len < 1024时,进行两倍扩容new.cap= 2(old.cap)
 3. 当旧切片的长度old.len > 1024时,进行1.25倍扩容new.cap = 1.25(old.cap)

我们知道,slice的空间大小等于元素size * cap,当旧切片的长度大于1024时,内存大小已经达到一个量级，如果还继续2倍扩容,那么消耗的内存空间将是非常大的。


接下来，在计算出新的容量的情况下，就需要准备去申请足够空间的内存，但之前还需要一系列内存对齐的计算操作：
当数组中元素所占的字节大小为1、8或者2的倍数时,对应相应的内存空间计算。
```go
func growslice(et *_type, old slice, cap int) slice {
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	/*
	 *lenmem表示旧切片实际元素长度所占的内存空间大小
	 *newlenmem表示新切片实际元素长度所占的内存空间大小
	 *capmem表示扩容之后的容量大小
	 *overflow是否溢出
	 */
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1://元素所占的字节数为1
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))//向上取整分配内存
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize://元素所占的字节数为8个字节
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size)://元素所占的字节数为2的倍数
		var shift uintptr
		//根据元素的字节数计算出位运算系数
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		//计算内存空间转化为用位运算
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}
}
```

计算出需要分配的内存大小后，就会重新申请内存,然后将原来切片的元素重新赋值到新的切片中。
```go
func growslice(et *_type,old slice,cap int)slice{
	var p unsafe.Pointer
	//如果元素不是指针
	if et.ptrdata == 0{
		//申请一块无类型的内存空间
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		//将超出切片当前长度的位置进行初始化
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	}else{
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		//根据元素类型申请内存空间
		p = mallocgc(capmem,et,true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
		}
	}
	//将旧切片的值拷入新的切片
	memmove(p, old.array, lenmem)
}
```

## 拷贝切片
拷贝切片可以用copy方法
> func copy(dst, src []Type) int
```go
func main() {
	var number1 =[]int64{1}
	var number2 =make([]int64,len(number1))
	copy(number2,number1)
	fmt.Println(number2)
}
```
实际上copy根据数据类型，最终会调用切片的runtime.slicecopy方法。
```go
func slicecopy(toPtr unsafe.Pointer, toLen int, fmPtr unsafe.Pointer, fmLen int, width uintptr) int {
	//如果源切片和目标切片长度为0,则直接返回0
	if fmLen == 0 || toLen == 0 {
		return 0
	}
	
	//根据源切片和目标切片的长度，以长度最小的切片进行拷贝
	n := fmLen
	if toLen < n {
		n = toLen
	}

	if width == 0 {
		return n
	}
	
	//拷贝的空间大小=长度 * 元素大小
	size := uintptr(n) * width
	if size == 1 { // common case worth about 2x to do here
		// TODO: is this still worth it with new memmove impl?
		//如果拷贝的空间大小等于1,那么直接转化赋值
		*(*byte)(toPtr) = *(*byte)(fmPtr) // known to be a byte pointer
	} else {
		//如果拷贝的空间大小大于1,则源切片中array的数据拷贝到目标切片的array
		memmove(toPtr, fmPtr, size)
	}
	return n
}
```
从源码可以看出在切片拷贝的时候，要预先定义切片的长度再进行拷贝，否则有可能拷贝失败。

## 总结
 1. 在扩容过程中,切片的地址不会被改变,改变的是切片的底层数组array,会申请一块新的内存地址替换。
 2. slice没有缩小容量的操作

