数组作为常见的数据结构，作为一组相同元素类型的集合,数组是一块固定大小的连续的内存空间。

## 结构体
数组的结构体在 [cmd/compile/internal/types.Array](https://github.com/golang/go/blob/da54dfb6a1f3bef827b9ec3780c98fde77a97d11/src/cmd/compile/internal/types/type.go#L340)
```go
type Array struct {
	Elem  *Type //元素类型 element type
	Bound int64 //元素的个数 number of elements; <0 if unknown yet
}
```

## 初始化
数组的初始化方式主要有两种,一种是定义指定大小的数值，第二种则是通过go的语法糖[...]
```go
[3]int{1,2,3}
[...]int{1,2,3}
```
第二种通过[...]初始化的方式,会在编译期间调用[cmd/compile/internal/gc.typecheckcomplit](https://github.com/golang/go/blob/da54dfb6a1f3bef827b9ec3780c98fde77a97d11/src/cmd/compile/internal/gc/typecheck.go#L2793)自动计算出数组的长度。

```go
func typecheckcomplit(n *Node) (res *Node) {
	.....
	if n.Right.Op == OTARRAY && n.Right.Left != nil && n.Right.Left.Op == ODDD {
		n.Right.Right = typecheck(n.Right.Right, ctxType)
		if n.Right.Right.Type == nil {
			n.Type = nil
			return n
		}
		//元素的类型
		elemType := n.Right.Right.Type
		
		//计算出元素的个数
		length := typecheckarraylit(elemType, -1, n.List.Slice(), "array literal")

		n.Op = OARRAYLIT
		//调用NewArray完成初始化
		n.Type = types.NewArray(elemType, length)
		n.Right = nil
		return n
	}
	.....
}
```

上面两种初始化过程,在go编译期间，最终都会调用NewArray来完成,并且一开始就确定该数组是会被分配在堆上还是在栈上,数组的初始化源代码在[cmd/compile/internal/types.NewArray](https://github.com/golang/go/blob/da54dfb6a1f3bef827b9ec3780c98fde77a97d11/src/cmd/compile/internal/types/type.go#L482)

```go
func NewArray(elem *Type, bound int64) *Type {
	if bound < 0 {
		Fatalf("NewArray: invalid bound %v", bound)
	}
	t := New(TARRAY)
	t.Extra = &Array{Elem: elem, Bound: bound}
	t.SetNotInHeap(elem.NotInHeap())
	return t
}
```

更多欢迎关注[go成神之路](https://github.com/friendlyhank/toBeTopgopher)



