大多数开发者认为，像Javascript这样的动态语言是没有类型的，但是事实上，ECMAScript语言中所有的值都有一个对应的语言类型，ECMAScript语言类型包括：Undefined、null、Boolean、String、Object、Number、symbol。
Javascript中的变量是没有类型的，只有值才有变量可以随时持有任何类型的值。
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
if(typeof atob === 'undefi*ned') {
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
