汇编的简单知识
go使用的汇编叫做plan9汇编。最初go是在plan9系统上开发的，后来才在Linux和Mac上实现。

## 寄存器
寄存器是CPU内部用来存放数据的一些小型存储区域，用来暂时存放参与运算的数据和运算结果。
```c
AX	累加寄存器(AccumulatorRegister)	用于存放数据，包括算术、操作数、结果和临时存放地址
BX	基址寄存器(BaseRegister)	用于存放访问存储器时的地址
CX	计数寄存器(CountRegister)	用于保存计算值，用作计数器
DX	数据寄存器(DataRegister)	用于数据传递，在寄存器间接寻址中的I/O指令中存放I/O端口的地址
SP	堆栈顶指针(StackPointer)	如果是symbol+offset(SP)的形式表示go汇编的伪寄存器；如果是offset(SP)的形式表示硬件寄存器
BP	堆栈基指针(BasePointer)	保存在进入函数前的栈顶基址
SB	静态基指针(StaticBasePointer)	go汇编的伪寄存器。foo(SB)用于表示变量在内存中的地址，foo+4(SB)表示foo起始地址往后偏移四字节。一般用来声明函数或全局变量
FP	栈帧指针(FramePointer)	go汇编的伪寄存器。引用函数的输入参数，形式是symbol+offset(FP)，例如arg0+0(FP)
SI	源变址寄存器(SourceIndex)	用于存放源操作数的偏移地址
DI	目的寄存器(DestinationIndex)	用于存放目的操作数的偏移地址
```

通用寄存器的名字在X64和plan9的对应关系:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200630223442527.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)

## 操作指令
用于指导汇编如何进行。以下指令后缀说明是64位上的汇编指令。
```c
#运算符号
ADDQ	运算	相加并赋值	ADDQ BX, AX表示BX和AX的值相加并赋值给AX
SUBQ	运算	相减并赋值	略，同上
IMULQ	运算	无符号乘法	略，同上
IDIVQ	运算	无符号除法	IDIVQ CX除数是CX，被除数是AX，结果存储到AX中
CMPQ	运算	对两数相减，比较大小	CMPQ SI CX表示比较SI和CX的大小。与SUBQ类似，只是不返回相减的结果

#数据传送指令
MOVQ	传送	数据传送	MOVQ 48, AX表示把48传送AX中
LEAQ	传送	地址传送	LEAQ AX, BX表示把AX有效地址传送到BX中

#压入和弹出栈数据
PUSHQ	传送	栈压入	PUSHQ AX表示先修改栈顶指针，将AX内容送入新的栈顶位置在go汇编中使用SUBQ代替
POPQ	传送	栈弹出	POPQ AX表示先弹出栈顶的数据，然后修改栈顶指针在go汇编中使用ADDQ代替

CALL	转移	调用函数	CALL runtime.printnl(SB)表示通过<mark>printnl</mark>函数的内存地址发起调用
JMP		转移	无条件转移指令	JMP 389无条件转至0x0185地址处(十进制389转换成十六进制0x0185)
JLS		转移	条件转移指令	JLS 389上一行的比较结果，左边小于右边则执行跳到0x0185地址处(十进制389转换成十六进制0x0185)
```
可以看到，表中的PUSHQ和POPQ被去掉了，这是因为在go汇编中，对栈的操作并不是出栈入栈，而是通过对SP进行运算来实现的。

## 标志位
```c
OF	溢出	0为无溢出 1为溢出
CF	进位	0为最高位无进位或错位 1为有
PF	奇偶	0表示数据最低8位中1的个数为奇数，1则表示1的个数为偶数
AF	辅助进位	
ZF	零	0表示结果不为0 1表示结果为0
SF	符号	0表示最高位为0 1表示最高位为1
```

## plain9函数定义
```c
// func add(a, b int) int
TEXT ·add(SB), NOSPLIT, $0-24
    MOVQ a+0(FP), AX // 参数 a
    MOVQ b+8(FP), BX // 参数 b
    ADDQ BX, AX    // AX += BX
    MOVQ AX, ret+16(FP) // 返回
    RET
```
为什么要叫TEXT?如果对程序数据在文件中和内存中的分段稍有了解的同学应该知道，我们的代码是在二进制文件中，是存储在.text段中，这里也就是一种约定俗成的起名方式。实际上plain9中TEXT是一个指令，用来定义一个函数。
定义中的 pkgname 部分是可以省略的，非想写也可以写上。不过写上 pkgname 的话，在重命名 package之后还需要改代码，所以推荐最好还是不要写。
中点 · 比较特殊，是一个 unicode 的中点，该点在 mac 下的输入方法是 option+shift+9 。在程序被链接之后，所有的中点 · 都会被替换为句号 . ，比如你的方法是 runtime·main ，在编译之后的程序里的符号则是 runtime.main 。嗯，看起来很变态。简单总结一下:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200630224244874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)


