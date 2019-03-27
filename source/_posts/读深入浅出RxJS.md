---
title: 深入浅出RxJS总结
---
很早就在一些博客和书籍上了解到，javascript是一门具备函数式编程特性的语言，虽然在日常工作也会接触到js的闭包，函数绑定，函数柯里化等概念，但对于函数式编程的特点和优势，一直一知半解。加之在工作中总会遇到异步处理的糟糕代码，所以前段时间花了一些功夫在阅读程墨著作的《深入浅出RxJS》上。总的来说，RxJS是Reactive Extension(也叫ReactiveX)编程理念的javascript版本。Reactive Extension是实践响应式编程的一套工具，其诞生的主要目的是解决异步处理的问题，但这并不表示Rx不适合同步的数据处理。这篇文章主要总结自己阅读《深入浅出RxJS》的一些思考，关于RxJS的具体使用可参考[ReactiveX](http://reactivex.io/)。
<!-- more -->

## 函数响应式编程
RxJS引用了两个重要的编程思想：函数式和响应式

### 函数式编程

函数式编程是强调使用函数来解决问题的一种编程范式。其对函数的使用有一些特殊的要求： 
* 声明式
和声明式编程相对的是命令式编程，命令式编程强调将计算逻辑以指令的方式描述出来：
```
function double(arr){
    const result = [];
    for(let i =0; i<arr.length; i++) {
        result.push(arr[i]*2);
    }
    return result;
}

```

    而声明式编程则把运算过程尽量写成一系列嵌套的函数调用：
    ```
    const double = arr => arr.map(item => item * 2);
    ```
    在javascript中，函数具有一等公民的地位，一个函数可以作为参数传递给另一个函数，才让map这种功能实现成为了可能。
* 纯函数
所谓纯函数，满足了以下条件
1. 函数的执行过程完全由输入参数决定，不会受除参数之外的任何数据影响.
2. 函数不会修改任何外部状态，比如修改全局变量或传入的参数对象.

* 数据不变性
程序要发生变化，不应该修改现有的数据，而是应该通过产生新的数据来体现这种变化。不可变的数据就是Immutable的数据。


  顺便对比一下函数式编程和面向对象编程。这两种编程方式都可以让代码更加容易理解，但方式不同。面向对象的方法把状态的改变封装起来，外部不能直接操作数据，只能通过类提供的实例方法来读取修改数据，限制了对数据的访问方式，这就制止了毫无节制的数据修改。但是面向对象却把数据的修改历史完全隐藏起来，这种不确定性导致代码可维护性下降。而函数式编程则是尽量减少发生变化的部分，数据就是数据，函数就是函数，函数可以处理数据，通过产生新的数据作为运算结果，以此让代码更加清晰。
  从本质上说，javascript并不是纯粹意义的函数式编程语言，javascript并没有强制要求数据不变性，编写的函数也不能保证没有副作用，但是javascript的函数拥有第一公民的身份，由此可以很方便地应用函数式编程的许多思想。
More info: 
[函数式编程入门教程](http://www.ruanyifeng.com/blog/2017/02/fp-tutorial.html)
[函数式编程初探](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)


### 响应式编程
>响应式编程是一种面向数据流和变化传递的编程范式。

数据流可以通过多种方式创造出来，流对象中流淌的是数据，数据流会通过各种管道，这些管道会对数据进行各种转化处理，RxJS的核心就是使用和组合各种操作符，构成管道，对流经其中的数据进行处理。

### Observable 和Observer
Observer和Obeservable是RxJS的两个重要概念。Obeservable就是“可被观察的对象”，而Observer就是观察者，连接两者的桥梁就是Oberservable对象的函数subscribe。
每个Observable对象，代表的就是在一段时间范围内发生的一系列事件。RxJs结合了观察者模式和迭代器模式，其中的Observable等同于：

``` 
Observable = Publisher + Iterator
```
* 观察者模式
观察者模式将逻辑分为发布者和观察者，其中生产者只负责产生事件，它会通知所有注册挂上号的观察者，而不关心这些观察者如何处理这些事件，相对的，观察者可以被注册上某个发布者，只管接收到事件之后就处理，而不关心这些数据是如何产生的。在RxJS的世界中，Observable对象就是一个发布者，通过Observable对象的subscribe，可以把这个发布者和某个观察者连接起来。
观察者使得复杂的问题被分解成三个小问题：
1. 如何产生事件，这是发布者的责任，在RxJS中是Observable对象的工作。
2. 如何响应事件，这是观察者的责任，在RxJS中由subscribe的参数来决定。
3. 什么样的发布者关联什么样的观察者，也就是何时调用subscribe。
More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/deployment.html)
