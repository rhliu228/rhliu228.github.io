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

## Observable 和Observer
Observer和Obeservable是RxJS的两个重要概念。Obeservable就是“可被观察的对象”，而Observer就是观察者，连接两者的桥梁就是Observable对象的函数subscribe。
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
* 迭代器模式
迭代器指的是能够遍历一个数据集合的对象，数据集合的实现方式有很多，比如数组，单向链表等，迭代器的作用就是提供一个通用的接口，让使用者完全不用关心这个数据集合的集合的实现方式。
迭代器的另一个名字叫游标（cursor）,就像一个移动的指针一样，从集合中的一个元素移动到另一个元素，完成对整个集合的遍历。
迭代器的实现方式有很多，通常包含以下几个函数： 
1. getCurrent，获取当前被游标指向的元素
2. moveToNext，将游标移动到下一个元素，调用这个函数之后，getCurrent获得的元素就会不同
3. isDone， 判断是否已经遍历完所有的元素
上面说的三个函数，是“拉”式迭代器的实现，而RxJs实现的是“推”式的迭代器的实现。因为在RxJS中，作为迭代器的使用者，并不需要主动从Observable中“拉”数据，而是只要subscribe上Observable对象之后，自然就能接收到消息的推送，这就是观察者模式和迭代器模式结合的强大之处。
下面是一段代码：

``` 
const iterator = getIterator();
while(!iterator.isDone()){
    console.log(iterator.getCurrent());
    iterator.moveToNext();
}

```
以下这段代码说明了Observable在完结，出错处理以及向下游传递数据时如何通知下游。
```
const onSubscribe = observer => {
    let number = 1;
    const handle = setInterval(() => {
        observer.next(number++);
    },1000);
    return {
        unsubscribe: () => {
            clearInterval(handle);
        }
    }
}
// 完结和出错处理
const onSubscribe = observer => {
    observer.next(1);
    // 向下游传递完结信号
    observer.complete();
    // or 向下游传递错误信号
    //注意，一个Observable对象只能有一种终结状态，要么是完结，要么是出错，所以调用了complete之后再调用error函数是无法引发下游的错误处理函数调用的，反之亦然。
    //observer.error('wrong');
}

const source$ = new Observable(onSubscribe);
const theObserver = {
    next: item => console.log(item),
    error: err => console.log(err),
    complete: () => console.log('No more Data)
}
const subscription = source$.subscribe(theObserver);
// 也可以简写为
// const subscription = source$.subscribe(
//    item => console.log(item),
//    err => console.log(err),
//    complete: () => console.log('No more Data)
//);
setTimeout(() => {
    //退订Observable
    subscription.unsubscribe();
},3500);
```
## 操作符
一个Observable对象代表的是一个数据流，实际工作中，产生Observable对象并不是每次都直接调用Observable构造函数来创建数据流对象，Rxjs已经贴心地为我们实现了常用的创建类操作符。这里说的创造，并不只是说返回一个Observable对象，而是指这些操作符不依赖于其他Observable对象，这些操作符可以凭空或者根据其他数据源（外部js事件，promise，ajax等）创造出一个Observable对象。事实上，对于复杂情况，并不会创建了一个数据流之后就直接subscribe一个Observer，往往需要各类操作符对数据流做一系列的处理，再交给Observer，就像一个管道，数据从管道的一段流入，途经管道的各个环节，位于管道末端的Observer只需要处理能够走到终点的数据。
  组成数据管道的元素就是操作符，操作符的本质是返回一个Observable对象的函数。对于每一个管道，链接的就是上游和下游。
  在数据管道中流淌的数据就像水，从上游流向下游。对于一个操作符来说，上游可能是一个数据源，也可能是其他操作符，下游可以是最终的观察者，也可能是另一个操作符。每个操作符都会满足
  1. 返回一个全新的Observable对象
  2. 对上游和下游的订阅和退订处理。
  3. 处理异常情况
  4. 及时释放资源

### 操作符分类
#### 功能分类
根据功能，操作符可以分为以下类别
* 创建类
* 转化类
* 过滤类
* 合并类
* 多播类
* 错误处理类
* 辅助工具类
* 条件分支类
* 数学和合计类

#### 静态和实例分类
  操作符还可以从存在形式进行分类，具体来说就是操作符的实现函数和Observable类的关系。对于定义在Observable类的静态函数，我们称之为静态操作符，而定义在由Observable类prototype属性指向的原型对象上的实例函数，则被称为实例操作符。在链式调用中，静态操作符只能出现在首位，而实例操作符可以出现在任何位置。有些操作符既可以作为Observable类的静态方法，又可以作为Observable对象的实例方法，比如merge。（此处涉及javascript的原型链知识以及es6的class，建议不太了解的读者查阅其他资料了解）

### 操作符的实现
#### 操作符函数实现

```
function map(project) {
    return new Observable(observer => {
        const sub = this.subscribe({
            next: value => {
                //处理异常情况
                try{
                    observer.next(project(value))
                } catch(err) {
                    observer.error(error);
                }
            },
            error: error => observer.error(error),
            complete: () => observer.complete()
        });
        //订阅和退订处理
        return {
            unsubscribe: () => {
                sub.unsubscribe();
            }
        }
    })
}
//使用es6箭头函数将会出错，因为此时this将直接绑定为定义函数环境下的this
//const map = (project) => {
    //这个函数体内的this并不是Observable对象本身
//}
```
可以看到，map利用new关键字创造了一个Observable对象，函数返回的结果就是这个对象，如此一来，map可以链式调用，可以在后面调用其他的操作符，或者调用subscribe增加Observer。这里的this代表的是上游的Observer对象，所以，可以直接使用subscribe订阅其中的事件，对于next事件，调用project函数，把推送的数据做映射，然后传递给下游，对于error和complete事件，map全部转手给下游处理。
#### 操作符关联Observable
map函数编写完毕之后，需要将这个函数与observable关联起来。
1. 给Observable打补丁
打补丁就像是给Observable类添加一点功能。
map操作符需要一个上游Observable对象，所以它是一个实例操作符，需要赋值给Observable的prototype：
```
Observable.prototype.map = mao;
```
如果是一个静态操作符，则直接赋给Observable类的某个属性。
2. 使用bind绑定Observable对象
有时候，我们并不希望一个操作符影响所有的Observable对象，为了不覆盖RxJS的map操作符，可以让自定义的操作符只对指定的Observable对象可用，这时可以用bind：
```
const result$ = map.bind(source$)(x => x*2);
```
也可以用call： 
```
const result$ = map.call(source$,x => x*2);
```
使用bind有一个缺点，就是上游Observable只能作为操作符函数的参数，这样没法用链式调用，比如，想要连续使用两个map：
```
const result$ = map.bind(map.bind(source$)(x => x*2))(x => x+1);
```
为了克服这个缺点，可以使用“绑定操作符”，绑定操作符是两个冒号，运算的时候绑定操作符后面的函数，但是保证函数运行时this是绑定操作符前面的对象，这样就可以使用链式调用：
```
const result$ = source$::map(x=>x*2)::map(x=>x+1);
```
绑定操作符并不是es6的标准语法，但它会出现在未来的es版本中，浏览器并不支持这种操作符。
3. 使用lift
RxJS v5对架构有很大的调整，很多操作符都会用一个神奇的lift函数实现，lift的含义是“提升”，功能是把Observable对象提升一个层次，赋予更多功能。lift是Observable的实例函数，它会返回一个新的Observable对象，通过传递给lift的函数参数可以赋予这个新的Observable对象特殊功能。
使用lift实现map： 
```
function map(project) {
    return this.lift(function(source$) {
        return source$.subscribe({
            next: value => {
                try{
                    this.next(project(value))
                } catch(err) {
                    this.error(error);
                }
            },
            error: err => this.error(err),
            complete: () => this.complete(),
        });
    });
}
```
this代表的是Observer对象，参数source$代表上游的Observable对象。
### 改进的操作符定义
#### 操作符和Observable关联的缺陷
js模块导入的代码并不都会被执行，打包工具（rollup，webpack等）的Tree shaking主要用于在js代码打包过程中去除无用的死代码，减少js代码的体积。
RxJS中操作符挂在Observable类上或者Observable.prototype上，赋值给Observable类和Observable.prototype上的某个属性在Tree shaking看来就是就是有用的代码，所以，所有的操作符，不管真实运行时是否被调用，都会被Tree shaking认为是有用的代码，不会被当作死代码删除。
除此之外，用给Observable打补丁的方式导入操作符，每个文件模块影响的都是全局唯一的那个Observable，极其容易因为代码耦合造成问题。
比如，代码引入interval和map两个操作符：
```
import 'rxjs/add/observable/interval';
import 'rxjs/add/operator/map';
```
假如在程序中interval和map并没有被调用过，这两个操作符也不会被当作死代码。
#### 使用call来创建库
摒弃给Observable类打补丁的做法，对于静态操作符，直接使用该函数即可，对于实例操作符，使用bind/call方法，让一个操作符只对一个具体的Observable对象生效。
```
//留意导入路径的不同
import {Observable} from 'rxjs/Observable';
import {of} 'rxjs/observable/of';
import {map} 'rxjs/operator/map';
Observable.prototype.double = function() {
    return this::map(x => x*2);
}
const source$ = of(1,2,3);
const result$ =source$.double();
result$.subscribe(value => console.log(value));
```
上述代码导入的of和map是两个独立的函数，RxJs传统的打补丁的方式，使用的也是observable和operator目录下的代码，例如rxjs/add/observable/of.js文件所做的工作就是导入rxjs/observable/of.js，并把导入的函数挂载到Observable类上：
```
"use strict";
var Observable__1 = require('../../Observable');
var of_1 = require('../../observable/of');
Observable__1.Observable.of = of_1.of;
```
使用bind和call方法，避免了Observable被污染的问题。
#### lettable和pipeable操作符
使用bind和call，每个函数体内依然需要访问this，访问this的函数并不是纯函数。另外，使用call也会让RxJS的代码失去类型检查的优势。从RxJS v5.5.0开始，加入了pipeable操作符，也曾称为lettable操作符。
1. let
在lettable操作符提出之前，let操作符就存在了，它接受一个函数作为参数，该函数需要接收一个Observable对象作为上游Observable。
```
import {Observable} from 'rxjs/Observable';
import 'rxjs/add/observable/of';
import 'rxjs/add/operator/map';
import 'rxjs/add/operator/let';
const source$ = Observable.of(1,2,3);
//double$是一个纯函数，map直接作用于参数obs$
const double$ = obs$ =>obs$.map(x=>x*2);
const result$ = source$.let(double$);
result$.subscribe(console.log);
```
改进以上代码，让map返回一个函数，从而可以作为let操作符的参数
```
function map(project) {
    return function(obs$){
        return new Observable(observer => {
            return obs$.subscribe({
                next: value => {
                    //处理异常情况
                    try{
                        observer.next(project(value))
                    } catch(err) {
                        observer.error(error);
                    }
                },
                error: error => observer.error(error),
                complete: () => observer.complete()
            });
        })
      
    }
}
const result$ = source$.let(map(x=>x*2));
```
let的作用是把map函数引入到链式调用之中，起到连接上游下游的作用。这里的map函数执行不再是返回一个Observable对象，而是返回一个函数，这个函数才返回Observable对象。，map的实现也看不到对this的访问，在数据管道中上游Observable对象以参数形式传入，而不是靠this获取，让map成为了一个纯函数。
从RxJS v5.5.0开始，加入了pipeable操作符，大部分操作符都有pipeable操作符实现，除了：
1. 静态操作符
2. 拥有多个上游Observable对象的操作符

因为每一个lettable操作符都是纯函数，且不会作为补丁挂在Observable类上，Tree shaking就能够找到根本不会被用到的操作符并将其去除。但是导入let这个操作符，却需要使用传统的打补丁的形式，所以RxJS让Observable类自带了一个新的操作符，名叫pipe，可以满足let的功能，却不需要像使用let一样导入模块，任何Observable对象都支持pipe
```
const result$ = source$.pipe(map(x=>x*2));
```
pipe还具有管道的功能，可以把多个lettable操作符连接起来：
```
import {of} 'rxjs/observable/of';
//留意lettable操作符的引入路径
import {map,filter} 'rxjs/operators';
const source$ = of(1,2,3);
const result$ = source$.pipe(
    filter(x => x%2 ===0),
    map(x => x*2)
);
result$.subscribe(console.log);
```
有四个操作符比较特殊，传统的操作符名称和pipeable操作符名称不同：
* do
* catch
* switch
* finally
这四个操作符名称都是js的关键字，以打补丁的方式赋值为Observable.prototype对象某个属性值没有问题，但是不能作为函数的标识符出现，这四个操作符对应的lettable操作符分别是
* tap
* catchError
* switchAll
* finalize