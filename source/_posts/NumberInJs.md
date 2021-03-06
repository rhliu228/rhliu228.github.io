---
title: 深入理解Javascript中的数字
date: 2019-04-02 19:05:00
#type: projects
tags: 
- Javascript
---

我们知道，在java中，数字会分为整型和浮点型，其中浮点型区分为单精度跟双精度格式。但是在JS中，只有Number型，并不区分整型跟浮点型，数字统一采用IEEE 754标准的64位双精格式进行存储。
<!-- more -->
#### 1. IEEE 754的双精度
IEE754浮点数有三个基本构成：符号域(S)、指数域(E)、尾数域(M)。给定数值V，用双精度浮点数表述为
```
V = (-1)^S×2^*(E-1023)*1.M
```
其中符号域S，指数E,尾数域M定义为： 
1. 符号域S：占1位 （0代表正数，1代表负数）
2. 指数E：占11位，也叫阶码（exponent），表示2的幂，它的作用是对浮点数加权。 阶码 = 阶码真值 + 偏移量
3. 尾数域(M)：占52位，M是二进制小数。

顺便提一下，32位单精度由1位符号位+8位阶码+23位尾数构成。
##### 1.1 为什么会有偏移量1023
11位指数表示范围为[-1024,1023], 需要引入符号位，例如将高位置1表示负数，这样0-1023表示正数，1024-2047表示负数。但这会给机器比较数字大小带来
麻烦（例如机器会认为2000比1023大，但实际上2000表示的是一个负数）。为了简化操作，可以考虑整体偏移1024位，变成[0,2047],要想得到原来的数字，只需要将存储数字减少1024即可。但由于数字0和2047适用于非规格化的情况，只能特殊处理（后面会介绍什么是规格化），去除了2个数字，所以用1023作偏移量即可。这种通过偏移，使得所有的数可以不用去考虑其符号的方法叫**余码系统**。经过以上处理，可以将指数的真实值称为阶码真值，阶码真值与偏移量相加得到阶码，阶码就是实际存储在机器上的数字。
##### 1.2 尾数M实际有多少位
同一浮点数的表示方法有很多种，但规范一般采用科学计数法，二进制只有0和1，那么按照科学计数法，首位只可能是1，对此IEEE省略了默认的1，所以实际上有效尾数是有53位的。这时会出现一个问题， 尾数M省略的1是一定会存在的，以至于无法表示0，不过IEEE 754早就想到了这个问题。
##### 1.3 E阶码取值
E阶码分为三种情况：
1. 规格化： S + (E!=0 && E!=2047) + 1.M。此时阶码不能为0也不能为2047，**只有这种情况，尾数域才会有隐含位1**。
2. 非规格化：此时E全为0，即阶码真值为-1023，如果尾数M全为0，则浮点数表示正负0；否则表示那些非常接近于0.0的数。
3. E全为1: 此时如果尾数域全为0，则表示Infinity和-Infinity，否则表示NaN：S + 11111111111 + (M!=0) 。

#### 2. 数字的范围
数字的范围有两个概念，一是最大正数和最小负数，二是最小正数和最大负数，即[最小负数，最大负数]并上[最小正数，最大正数]。从S、E、M三个维度看，S代表正负，E阶码值远大于M尾数个数，所以S决定大小，M决定精度。下面以E阶码分两种情况分析：
##### 2.1 规格化
规格化下，当E取最大值，即2046时，阶码真值为2046-1023=1023，从指数上看，数值范围是[-2^1023,2^1023]。JS函数计算Math.pow(2,1023)的结果是8.98846567431158e+307，如果尾数全为1，即1.1111111111 1111111111 1111111111 1111111111 1111111111 11，非常接近于2，将8.98846567431158e+307乘以2，得到的结果约等于1.7976931348623157e+308。这个值就是我们用JS常量**Number.MAX_VALUE**获取到的，两者非常接近，所以数字的范围是[-1.7976931348623157e+308,1.7976931348623157e+308]。如果数字绝对值超过1.7976931348623157e+308，则数字太大或者太小，在JS显示为Infinity和-Infinity，称为**正向溢出**。
##### 2.2 非规格化
非规格化的情况下，E取值为0，阶码真值为-1023，指数最小值是2^1023，然而尾数等于0.0000000000 0000000000 0000000000 0000000000 0000000000 01，52位尾数还能虚拟化地向右移动51位，所以最小值是2^(-1074) = Math.pow(2,2074) 约等于5e-324。JS常量**Number.MIN_VALUE**就等于5e-324。所以(-5e-324,5e-324)之间的数比可表示的最小的数还要小，叫**反向溢出**。
#### 3.JS 整数的安全范围
从M尾数分析，精度最多是53位，这决定了整数的安全范围远远小于Number.MAX_VALUE。如M取最大值：1.1111111111 1111111111 1111111111 1111111111 1111111111 11，E取52，则得到的结果是2^53-1,得到的结果是9007199254740991。而在ES6中，能够被“安全”呈现的最大整数是**Number.MAX_SAFE_INTEGER**，等于9007199254740991；最小整数是-9007199254740991，在ES6被定义为**Number.MIN_SAFE_INTEGER**。

#### 4. 较小的数值比较
二进制浮点数的最大问题，是会出现以下情况： 
```
0.1 + 0.2 === 0.3  // false
```
因为尾数M只有52位，这决定了0.1和0.2不是十分精确，它们的相加结果并非刚好等于0.3，而是一个比较接近的数字0.30000000000000004，所以条件判断为false。为了判断0.1+0.2和0.3是否相等，最常见的方法是设置一个误差范围值，通常称为“机器精度”，对于JavaScript来说，这个值通常是2^-52(2.220446049250313e-16)。从ES6中开始，这个值被定义在**Number.EPSILON**中。

#### 5. 32位有符号整数
虽然整数最大能够达到53位，但是有些数字操作（如数位操作）只适用于32位数字，所以这些操作中数字的安全范围就要小很多，变成从Math.pow(-2,31)(-2147483648)到Math.pow(2,31)(2147483648)。例如 a|0 可以将a中的数值转换为32位有符号整数

#### 6. 32位无符号整数
Javascript数组下标值的范围为0到2^32-1。对于任意给定的数字下标值，如果不在此范围内，js会将它转换为一个字符串，并将该下标对应的值作为该数组对象的一个属性值而不是数组元素。如果该下标值在合法范围内，则无论该下标值是数字还是数字字符串，都一律会被转化为数字使用，即 array["0"] = 0 和 array[0] = 0 执行的是相同的操作。

#### 7. 总结
用数轴来表示JS中各个Number常量的大小如下：

![JS中的数轴](/assets/img/axis.png)

|  |JS 32位整数 | 64位双精度 |
| :------|:------| :------: |
| 场景 | 数组索引，位运算 | Number |
| 整数范围 | 数组索引：[0,2^32-1]，位运算：[-2^31,2^31] | [-2^53-1,2^53-1] |
| 可表示的数范围 | 同上 | [-1.7976931348623157e+308,-5e-324]U [5e-324, 1.7976931348623157e+308] |
| 精度 | 1 | 2^-52|

参考
* [深入理解IEEE 754的64位精度](https://www.boatsky.com/blog/26)
