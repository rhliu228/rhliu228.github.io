---
title: JS的类型和特殊值
date: 2019-06-22 09:01:00
tags: 
- Javascript
---
大多数开发者认为，像Javascript这样的动态语言是没有类型的，但是事实上，ECMAScript语言中所有的值都有一个对应的语言类型，ECMAScript语言类型包括：Undefined、null、Boolean、String、Object、Number、symbol。
Javascript中的变量是没有类型的，只有值才有。变量可以随时持有任何类型的值。
<!-- more -->
##### 1. undefined和undeclared
已在作用域中声明但是还没有赋值的变量，是undefined的；相反，还没有在作用域中声明过的变量，是undeclared的。
```
var a;
a;   // undefined
b;   // ReferenceError: b is not defined
```

这里 b is not defined 很容易让人以为是“b is undefined”,这里显示为为“b is not declared”会更准确。
更让人抓狂的是typeof处理undeclared变量的方式：
```
var a;
typeof a;   // undefined
typeof b;   // undefined
```
对于undeclared的变量，typeof居然返回“undefined”。b虽然是一个undeclared的变量，但是typeof b并没有报错，这是因为typeof有一个特殊的安全防范机制。
然而，typeof的安全机制对于在浏览器中运行的js代码来说还是很有用处的，例如：
```
// 这样会报错
if(DEBUG){
    console.log('debug mode');
}

// 这样是安全的
if(typeof DEBUG !== 'undefined) {
    console.log('debug mode');
}
```

还有要为某个缺失的功能编写polyfill：
```
if(typeof atob === 'undefined') {
    //这里没有用var声明变量
    atob = function() {/** */}
}
```

如果在if语句里声明var atob，因为存在变量声明提升，该声明会被提升到作用域最顶层，即使if语句条件不成立也是如此（即浏览器本来就支持atob）。在某些浏览器中，对于特殊的内置全局变量（宿主对象），这样的重复声明会报错。
还可以通过判断某个全局变量是否是全局对象的属性来避免undeclared变量的判断出错：
```
if(!window.atob) {

}
```

与undeclared变量不同，访问不存在的对象属性不会产生Reference error。

##### 2. 字符串
JS中的字符串是不可变的，但是数组是可变的。字符串不可变是指字符串的成员函数不会改变其原始值，而是创建并返回一个新的字符串，而数组的成员函数都是在其原始值上进行操作。
可以借用数组的非变异函数来处理字符串：
```
var a = 'foo';
var c = Array.prototype.join.call(a,"-");  //f-o-o
var d = Array.prototype.map.call(a,function(v){
    return v.toUpperCase() + ".";
});   //"F.O.O."

```

但是无法借用数组的变异函数来处理字符串，例如reverse，push等。为了实现字符串反转，可以：
```
var c = a.split("").reverse().join("");
```
JS的数组变异函数包括： 
* push()
* pop()
* shift()
* unshift()
* splice()
* sort()
* reverse()

##### 3. undefined和null
undefined类型只有一个值，即undefined，null类型也只有一个值，即null；它们的名称既是类型也是值。它们之间有一些细微的区别：
* null指空值，undefined指没有值
* undefined指从未赋值，null指曾经赋过值，但目前没有值

null是一个特殊关键字，不是标识符，不能被当作变量来使用和赋值，而undefined是一个标识符，可以被当作变量来使用和赋值。
3.1 在非严格模式下，我们可以为全局标识符undefined赋值，
```
undefined = 2;   //ugly

"use strict";
undefined = 2;  //TypeError
```
3.2 在非严格和严格模式下，可以声明一个局部变量undefined：
```
var undefined = 2;
console.log(undefined);  //ugly
```
3.3 undefined是一个内置标识符，它的值为undefined，通过void运算符即可得到该值。
```
var a = 2;
console.log(void a, a);  //undefined 2
```
利用void运算符可以让表示式不返回任何结果，例如用于函数return语句中。
```
function foo() {
    //...
    return void setTimout(dosomething, 500);
}
```
##### 4. NaN
js中有一个特殊的数字：NaN。NaN的含义是“Not a number”，如果数字运算的操作数不是数字，就无法返回一个有效的数字，这种情况下应该返回NaN。
```
var a = 2/'foo';  // NaN
typeof a === 'number';  // true
```
NaN是唯一一个非自反的值，即NaN !== NaN 为true。既然无法对NaN进行比较，那如何判断一个值是否是NaN呢?可以利用window对象内置的isNaN()方法。
```
isNaN(a);  //true
```
但是这样方法有一个缺陷，它的检查方式是检查参数是否不是NaN，也不是数字。
```
var a = 2/'foo';
var b = 'foo';
window.isNaN(a);  // true
window.isNaN(b);  // true
```
从ES6开始可以用Number.isNaN()检测，这个方法会避免上面的bug，其polyfill如下： 
```
if(!Number.isNaN) {
    Number.isNaN = function(n) {
        return typeof n === 'number' &&
        window.isNaN(n);
    }
}
//更简单的方法
if(!Number.isNaN) {
    Number.isNaN = function(n) {
        return n!==n;
    }
}
```

##### 5. 零值
Javascript中有一个常规的0（也叫做+0）和-0。
-0除了可以用作常量之外，也可以是某些数学运算的返回值：
```
var a = 0 / -3; //-0
var b = 0 * -3; //-0
```
加法和减法运算不会得到-0。有时候数学运算的符号位用来表示移动方向等信息。此时如果一个值为0的变量失去了它的符号位，它的方向信息就会丢失，这是-0存在的意义。
根据规范，对负零进行字符串操作会返回“0”：
```
var a = 0 / -3;
console.log(a);  // "-0"

//但是规范定义的返回结果是这样：
a.toString(); //"0"
a+"";         //"0"
String(a);    //"0"
JSON.stringify(a); //"0"
```
此外， 0与-0进行===比较时会返回true，为了区分二者，需要做一些特殊处理：
```
function isNegZero(n) {
    n = Number(n);
    return (n===0) && (1/n===-Infinity)
}
```

##### 6. 特殊等式
NaN和-0在相等比较时表现特殊，es6加入了Object.is()来判断两个值是否绝对相等，可以用来处理上述的特殊情况：
```
if(!Object.is) {
    Object.is = function(v1, v2) {
        //判断是否是-0
        if(v1===0 && v2===0){
            return 1/v1 === 1/v2;
        }
        //判断是否是NaN
        if(v1 !== v1) {
            return v2 !== v2;
        }
        return v1 === v2;
    }
}
```
注意，能使用==和===就不要使用Object.is(),因为前者效率更高，更为通用。Object.is主要用来处理那些特殊的相等比较。

##### 7. 值和引用
在许多编程语言中，赋值和参数传递可以通过值复制或者引用复制的方法来完成，取决于使用的语法。但是在JS中，对值和引用的复制完全根据值的类型来决定，在语法上没有任何区别:
**简单值总是通过值复制的方式来赋值/传递；复合值总是通过引用复制的方式赋值或者传递。**
例如：
在c语言中
```
void swap(int &x, int &y) {
    int temp;
    temp = x;
    x = y;
    y = temp;
}
main() {
    int a =1, b =2;
    swap(a,b);
    cout<<"a="<<a<<"b="<<b<<"\n";
    return 1;
}
//输出 a=2 b=1
```
传引用调用实际上在形参位置插入的是变量本身，即值的内存位置。由于程序变量是作为内存位置来实现的，所以这些内存位置就是变量。引用相当于变量的**别名**，本身并不单独分配自己的内存空间，二者指向的是同一处内存。
而在JS中：
```
function foo(x) {
    x.push(4);
    x; //[1,2,3,4]
    x = [4,5,6];
    x.push(7);
    x; //[4,5,6,7]
}
var a = [1,2,3];
foo(a);
a; //[1,2,3,4]
```
向函数传递a的时候，实际是将引用a的复本赋值给x，刚开始x和a指向的是同一处内存，所以x.push(4)改变的是同一个值。随后x不再指向数组[1,2,3,4],而是指向新分配的内存[4,5,6]。因为数组是复合值，所以这里默认采用引用赋值，不需要特殊的语法。