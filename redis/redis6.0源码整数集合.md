整数集合是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合键的底层实现。
```c
typedef struct intset {
	//编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
} intset;
```

 contents数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数据项(item),各个项在数组中按值得大小从小到大有序得排列，并且数组中不包含任何重复项。
 
 length属性记录了整数集合包含得元素数量，也即是contents数组得长度。

虽然intset结构将contents属性声明为int8_t类型的数组，但实际上contents并不保存任何int8_t类型的值，contents数组得真正类型取决于encoding属性的值:

 - 如果encoding属性得值为INTSET_ENC_INT16，那么contents就是一个int16_t类型的数组，数组里得每个项都是一个int16_t类型的整数值(最小值为-32768,最大值为32767)。
 - 如果encoding属性得值为INTSET_ENC_INT32，那么contents就是一个int32_t类型的数组，数组里的每个项都是一个int32_t类型的整数值(最小值为-2147483648,最大值为2147483648)。
 - 如果encoding属性的值为INTSET_ENC_INT64,那么contents就是一个int64_t类型的数组，数组里的每个项都是一个int64_t类型得整数值(最小值为-9223372036854775808,9223372036854775808)

## 创建一个新得整数集合
```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```

## 插入
```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
	//获取值的长度大小
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    //默认设置插入成功
    if (success) *success = 1;

   //当值的长度大于当前编码
    if (valenc > intrev32ifbe(is->encoding)) {
        //进行升级操作并插入元素
        return intsetUpgradeAndAdd(is,value);
    } else {
        //二分查找当前元素,看是否存在
        //如果存在,那么将*success置为0，并返回未经修改的整数集合
        //如果不存在,那么可以插入value的位置将被保存在pos指针中
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }
		//重新调整集合的空间大小
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        //如果新元素不是被添加到底层数组的末尾
        //则对现有的元素进行移动,空出pos上的位置,用于设置新值
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
	// 将新值设置到底层数组的指定位置中
    _intsetSet(is,pos,value);
    //修改整数集合的长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

## 升级(不可逆)
```c
/* Upgrades the intset to a larger encoding and inserts the given integer.
 *
 * 根据值 value 所使用的编码方式，对整数集合的编码进行升级，
 * 并将值 value 添加到升级后的整数集合中。
 *
 * 返回值：添加新元素之后的整数集合
 *
 * T = O(N)
 */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
	//当前的编码方式
    uint8_t curenc = intrev32ifbe(is->encoding);
    //新值所需的编码方式
    uint8_t newenc = _intsetValueEncoding(value);
    //当前集合的元素数量
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    //设置新的编码，并且重新分配内存空间
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

   //将所有元素从旧的编码集合转换到新的编码集合里
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    //值要么在头部插入要么在尾部插入
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    //更新整数集合的元素数量
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

下面重点说一下升级的步骤，共分为三步进行：

 1. 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
 2. 将底层数组现有的所有元素转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。
 3. 将新元素添加到底层数组里面。

举个例子，假设现在有一个INTSET_ENC_INT16编码的整数集合，集合中包含三个int16_t类型的元素。
因为每个元素都占用16位空间，所以整数集合底层的数组大小为3*16=48位。
现在，假设我们要将类型为int32_t的整数值65535添加到整数集合里面，因为65535的类型int32_t比整数集合当前所有元素的类型都要长，所以在这之前要进行升级。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730080904507.png)

 - 整数集合目前有三个元素，再加上新元素65535,整数集合需要分配四个元素的空间，因为每个int32_t整数值需要占用32位空间，所以空间重分配之后，底层数组的大小将是32*4=128位,如图所示，虽然程序对底层数组进行了空间重分配，但数组原有的三个元素1、2、3仍然是int16_t类型，这些元素还保存在前48位里面，所以程序接下来做的就是将这三个元素转换成int32_t类型，并将转换后的元素放置到正确的位上面，而且在放置元素的过程中，需要维持底层数组的有序性质不变。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730082816749.png)
 - 从最后一位原有集合元素开始，调整每个原有元素转换成int32_类型，使其转换到正确的位上。
调整元素3大小：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730084159472.png)
调整元素2大小:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730084235947.png)
调整元素1的大小
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730084257687.png)
元素65535在1、2、4、65535四个元素中排名第四，所以被添加到contents数组的索引3位置上，最后，程序将整数集合encoding属性的值从INTSET_ENC_INT16改为INTSET_ENC_INT32，并将length属性的值从3改为4。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730090731155.png)

注意：因为引发升级的新元素的长度总是比整数集合现有所有元素的长度都大，所以这个元素的值要么大于所有现有元素，要么小于所有现有元素(负数)

 1. 新元素小于所有现有元素的情况下，现有元素在升级的时候会在原所在索引上+1,空出头部索引0插入新元素
 2. 新元素大于所有现有元素的情况下，现有元素在升级的时候索引保持不变，新元素会在尾部插入。
 
 

## 其他
```c
//重新调整集合的空间大小
static intset *intsetResize(intset *is, uint32_t len) {
    uint32_t size = len*intrev32ifbe(is->encoding);
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}
```

```c
//根据给定的编码方式,返回集合的底层数组再pos索引上的元素
//T = O(1)
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64);
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}
```

```c
//根据集合的编码方式，将底层数组在 pos 位置上的值设为 value
//T = O(1)
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```

