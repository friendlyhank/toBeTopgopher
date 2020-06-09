## 数值类型
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200519181002348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
**浮点数和定点数：**
对于小数的表示，MSQL分为两种方式：浮点数和定点数。浮点数包括float(单精度)和double(双精度),而定点数只有decimal一种表示。定点数在MySQL内部以字符串形式存放，比浮点数更精确，适合用来表示货币等精度高的数据。
浮点数和定点数都可以用类型名称后加"(M,D)"的方式进行表示，"(M,D)"表示该值一个共显示M位数字(整数位+小数位),其中D位位于小数点后面，M和D又称为精度和标度。
浮点数在插入的时候会根据精度和标度自动四舍五入后的结果插入，系统不会报错；定点数如果不写精度和标度，则按照默认值decimal(10,0)来进行操作，并且如果数据超越了精度和标度值，系统则会报错。
## 日期和时间类型
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020051918093283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
 - 如果要用来表示年月日，通常用DATE来表示。
 - 如果用来表示年月日时分秒，通常用DATETIME表示。
 - 如果用来表示时分秒，通常用TIME来表示。
 - 如果需要经常插入或者更新日期为当前系统时间，则通常用TIMESTAMP来表示。
 - 如果只表示年份，可以用YEAR来表示。

## 字符串类型
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200519180916356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
由于CHAR是固定长度的，所以它的处理速度比VARCHAR快得多，但是其缺点是浪费存储空间。

参考：[https://www.runoob.com/mysql/mysql-data-types.html](https://www.runoob.com/mysql/mysql-data-types.html)

更多讲解,欢迎关注我的github:
[go成神之路](https://github.com/friendlyhank/toBeTopgopher)