---
title: 深入浅出RxJS总结
date: 2019-03-15 16:01:00
tags:
- 函数式编程
- 响应式
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
* 创建类，包括create, of, range, generate, repeat和repeatWhen, empty, throw, never, inteval和timer, from, from Promise, from event和fromEventPattern, ajax, defer.
* 转化类，包括map, mapTo, pluck, windowTime、 windowCount、windowWhen、windowToggle、和window, bufferTime、bufferCount、bufferWhen、bufferToggle和buffer, concatMap、mergeMap、switchMap、exhaustMap, scan和mergeScan
* 过滤类，包括filter, first, last, take, takeLast, takeWhile和takeUntil, skip, skipwhile和skipUntil, throttleTime、debouceTime和auditTime, throttle、debouce和audit, sample和sampleTime, distnct, distinctUntilChanged和distinctUntilKeyChanged, ignoreElements, elementAt, single
* 合并类，包括concat和concatAll, merge和mergeAll, zip和zipAll, combineLatest、combineAll和withLatestFrom, race, startWith, forkJoin, switch和exhaust
* 多播类，包括multicast, publishLast, publishReplay, publishBehavior
* 错误处理类，包括catch, retry和retryWhen, finally
* 辅助工具类，包括concat, max和min, Reduce, every, find和findIndex, isEmpty, defaultEmpty

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
比如，代码引入interval和map两个操作符：
```
import 'rxjs/add/observable/interval';
import 'rxjs/add/operator/map';
```
假如在程序中interval和map并没有被调用过，这两个操作符也不会被当作死代码。
除此之外，用给Observable打补丁的方式导入操作符，每个文件模块影响的都是全局唯一的那个Observable，极其容易因为代码耦合造成问题。

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

## 多播
在RxJs中，Observable和Observer的关系，就是前者在播放内容，后者在收听内容，播放内容的方式可以分为三种：
* 单播（unicast）
* 广播（broadcast）
* 多播（multicast）
单播是一对一的关系，一个播放者对应一个接听者，广播把消息传播给所有接听者，多播则是有选择性地把消息传递给有需要的接听者。RxJS对单播是绝对支持的，而广播则不是RXJS支持的目标，广播已经有很多现成的解决方法，例如nodeJs中的EventEmitter。

### Hot和Cold数据流的差异
如果每一次观察者对Observable对象进行subscribe，都会产生一个全新的数据序列的数据流，这样的Observable对象被称为cold observable。RxJS的大部分创建类操作符创建出来的都是cold observable对象，例如inteval，range等。
下面是一个单播的例子：
```
import {interval} 'rxjs/observable/of';
import {take} 'rxjs/operators';
const tick$ = interval(1000).pipe(take(3));
tick$.subscribe(value=>console.log('observer 1: ' + value));
setTimeout(() => {
    tick$.subscribe(value=>console.log('observer 2: ' + value));
},2000);

//console
//observer 1: 0
//observer 1: 1
//observer 2: 0
//observer 1: 2
//observer 2: 1
//observer 2: 2 
```
你可能会以为输出下面的结果：
```
//observer 1: 0
//observer 1: 1
//observer 2: 1
//observer 1: 2
//observer 2: 2 
```
但是，interval操作符产生的是一个cold observable对象，每次对上游的subscribe都会产生一个新的生产者。
而对于一个hot observable，概念上有一个独立于Observable对象的生产者，这个生产者的创建与subscribe的调用没有关系，subscribe的调用只是让Observable对象连接上生产者而已。RxJs中有一些操作符产生的是Hot Observable：
* fromPromise
* fromEvent
* fromEventPattern

这些产生hot observable对象的操作符数据源都在外部，真正的数据源和有没有Observer没有任何关系。而真正的多播，则是不管有多少Observer进行subscribe，推给Observer的数据都是一样的数据源，满足这种条件的，就是hot observable。
hot observable和cold observable都具有“懒”的性质，两者的数据管道内逻辑都只有订阅者存在时才执行，但是cold Observable更“懒”，如果没有订阅者，连数据都不会真正产生；对于hot observable来说，没有订阅者的情况下，数据依旧产生，只是不传入数据管道。
所以cold observable实现的是单播，而hot observable实现的是多播。
### Subject
有时候，我们也希望对cold observable实现多播。要把一个cold observable对象转换成一个hot observable，并不是去改变cold observable本身，而是产生一个新的observable对象，包装之前的cold observable对象，这样在数据流管道中，新的observable就成为了下游。
要实现这个转化，很明显需要一个“中间人”做串接的事情：
* 中间人需要提供subscribe方法，让其他人能够订阅自己的数据源
* 中间人能够有办法接收推送的数据，包括cold observable推送的数据
RxJS中，提供了subject类型，subject既有observable的接口，也具有observer的接口。
```
import {Subject} from 'rxjs/Subject';
import {interval} 'rxjs/observable/interval';
import {map} 'rxjs/operators';
const subject = new Subject();
subject.pipe(map(x=>x*2)).subscribe(
    value => console.log(value),
    err => console.log(err),
    () => console.log('on complete')
);
subject.next(1);
subject.next(2);
subject.complete();
```
一个subject对象是一个Observable，所以可以在后面链式调用任何操作符，也可以调用subscribe来添加Observer。
一个subject对象同时也是一个Observer，所以也支持next，error和complete方法。

### 用Subject实现多播
```
import {Subject} from 'rxjs/Subject';
import {interval} 'rxjs/observable/interval';
import {map} 'rxjs/operators';
const tick$ = interval(1000).pipe(take(3));
const subject = new Subject();
tick$.subscribe(subject);
subject.subscribe(value=>console.log('observer 1: ' + value));
setTimeout(() => {
    subject.subscribe(value=>console.log('observer 2: ' + value));
},1500);
```
只需要让Subject对象居于cold observable和observer之间。
但是很可惜subject并不是一个操作符，所以无法链式调用，不过可以创建一个新的操作符来达到链式调用的效果:
```
Observable.prototype.makeHot = function() {
    const cold$ = this;
    const subject = new Subject();
    cold$.subscribe(subject);
    return subject;
}
const hotTick$ = interval(1000).pipe(take(3)).makeHot();
hotTick$.subscribe(value=>console.log('observer 1: ' + value));
setTimeout(() => {
    hotTick$.subscribe(value=>console.log('observer 2: ' + value));
},1500);
```
这段代码有个漏洞，可以直接调用makeHot返回的subject对象的next，error或者complete方法来影响下游：
```
const hotTick$ = interval(1000).pipe(take(3)).makeHot();
hotTick$.complete();
//下面的Observer将不会收到任何消息
hotTick$.subscribe(value=>console.log('observer 1: ' + value));
```
subject对象是不能重复使用的，一个subject对象一旦被调用了complete或者error函数，那么，它作为observable的生命周期也就结束了，后续再想利用这个subject对象传递数据给下游，就像泥牛如大海，没有任何反应。
为了杜绝这种可能性，对makeHot进行改进，让它返回一个纯粹的Observable对象:
```
Observable.prototype.makeHot = function() {
    const cold$ = this;
    const subject = new Subject();
    cold$.subscribe(subject);
    return Observable.create((observer) => subject.subscribe(observer));
}
```
makeHot并不是直接返回Subject对象，而是返回一个新的Observable对象，这样就避免了subject直接暴露给外部。

subject可以有多个上游，如果一个subject订阅多个数据流，起到的作用就是把多个数据源的内容汇聚到一个observable，但是这种使用方式却可能引发意想不到的结果。假设其中一个上游调用了subject对象的complete函数，那即使其他上游的数据还没推送完，subject也会因为生命周期的结束，无法再把其他数据推送给下游。
任何一个上游数据的完结或者出错都可以终结subject对象的生命，让subject来做合并数据流的工作并不合适，应该让merge来做。
当subject有多个observer时，如果某个observer产生了一个错误异常，而且这个异常没有被observer处理，那subject的其他observer都会失败。
```
const tick$ = interval(1000).pipe(take(3));
const subject = new Subject();
tick$.subscribe(subject);
const throwOnUnluckyNumber = value => {
    if(value==4){
        throw new Error('unlucky number 4');
    }
    return value;
}
subject.pipe(map(throwOnUnluckyNumber)).subscribe(
    value=>console.log('observer 1: ' + value)
)
subject.subscribe(
    value=>console.log('observer 2: ' + value),
    err=>console.log(err)
)
```
1号observer在遇到数字4的时候遇到错误异常，2号observer因为1号observer没有优雅地处理错误，也被牵连，因为subject对象由于下游1号Observer没有处理错误而被破坏了。
可以想象，Subject为了给所有observer推送数据，会有类似的代码：
```
for(let observer of allObservers){
    observer.next(data);
}
```
为了解决这个问题，好的编程实践是让所有的observer都具备对异常错误的处理。
```
subject.pipe(map(throwOnUnluckyNumber)).subscribe(
    value=>console.log('observer 1: ' + value),
    err=>console.log('observer 1 on error: ' + err)
)
subject.subscribe(
    value=>console.log('observer 2: ' + value),
    err=>console.log('observer 2 on error: ' + err)
)
```
### 支持多播的操作符
RxJs提供了支持多播的一系列操作符，其中最基础的是
* multicast
* share 
* publish
1. multicast是一个实例操作符，能够以上游的Observable为数据源产生一个新的hot observable对象：
```
const hotSource$ = coldSource$.multicast(new Subject);
```
multicast接收一个subject对象或者一个返回subject对象的函数（可以在subject对象生命终结时重新subscribe上游）作为参数，
返回的是一个Observable对象，不过这个对象比较特殊，是Observable子类ConnectableObservable的实例对象。这种对象包含一个connect函数，connect的作用是
触发multicast用Subject对象去订阅上游的Observable，如果不调用这个函数，这个ConnectableObservable将不会从上游那里得到任何数据。
除此之外，ConnectableObservable还支持自动计数，对Observer的个数进行计数，当第一个Observer对象被添加时，主动去订阅上游，当最后一个Observer退订时，就让中间人
Subject退订上游的Cold Observable。这个功能可以借助ConnectableObservable对象的函数refCount实现。
除了第一个参数指定一个Subject对象或者指定一个产生Subject对象的工厂方法，multicast还支持第二个参数：selector，这个参数是一个可选参数，它可以使用上游数据任意多次，但不会重复订阅上游数据流。
一旦指定selector参数，multicast将不会返回ConnectableObservable对象，而是用selector函数来产生一个Observable对象。
selector函数有一个参数shared，这个参数就是multicast第一个参数代表的Subject或者使用工厂方法返回的Subject对象。
```
const coldSource$ = interval(1000).pipe(take(3));
const selector = shared => {
    return shared.pipe(concat(of('done)));
}
const tick$ = coldSource$.pipe(multicast(new Subject(),selector));
tick$.subscribe(
    value=>console.log('observer 1: ' + value),
    err=>console.log('observer 1 on error: ' + err)
);
setTimeout(() => {
    tick$.subscribe(
        value=>console.log('observer 2: ' + value),
        err=>console.log('observer 2 on: error: ' + err)
    );
});
//console
//observer 1: 0
//observer 1: 1
//observer 1: 2
//observer 1: done
//observer 2: done
```
2. publish完全是通过multicast来实现的
```
function(selector){
    if(selector){
        return this.multicast(()=>new Subject(),selector);
    }else{
        return this.multicast(new Subject);
    }
}
```
3.  shared完全是通过multicast来实现的
```
Observable.prototype.shared = function shared() {
    return this.multicast(() => new Subject()).refCount();
}
```
除了以上这几个基础的多播操作符外，RxJS还支持三个高级多播操作符：
* publishList
* publishReplay
* publishBehavior
关于它们的使用可以自行查阅官网

## 总结
《深入浅出RxJS》这本书较为系统的介绍了RxJS的核心特性和各类操作符，并且书该书结合弹珠图，将代码例子形象生动地描述清楚，给读者提供了很好的入门教程。个人觉得从书中前三章中得到的收获最大，从中了解到RxJS的代码架构、函数式编程的理念和如何实现操作符。后面介绍操作符用途的章节略显冗长，不过其中有些代码的例子特别贴切和生动，让人有眼前一亮的快感。可以看出作者对RxJS各类操作符的了解是非常深刻的。
  例如，讲解RxJS的高阶observable，所谓高阶，指的是该Observable返回的依旧是Observable，这样能够管理多个数据流，将管理数据和管理数据流归一化，类似于高阶函数，高阶函数的参数或者返回值是一个函数。这时就需要一些操作符能够组合或者处理这些高阶Observable，将其“砸平”，常见的有合并类高阶操作符concatAll，mergeAll，以及高阶的map等。
为了说明高阶map运算符concatMap的用途，作者列举了实现网页拖拽的例子。网页应用中，拖拽就是用户的鼠标在某个dom元素上按下去，然后拖动这个元素，最后松开鼠标的过程，这个过程是重复的，拖拽涉及的事件包括
mousedown，mouseup和mousemove，使用传统方式，基本上就是当mousedown事件发生时，用一个变量标识当前进入拖拽状态，然后监听mousemove事件，移动dom元素位置，当mouseup事件发生时，改变状态变量使之标记为“离开拖拽”，等待下一次mousedown事件的发生。
这个过程可以看成是多个由mousedown事件引发的数据流序列，每个序列内部又是以mouseup结束的mousemove数据序列。这些序列相互之间不可能交叉重复，这时可以考虑使用高阶map操作符concatMap实现这个例子。详细的代码可参考[concatMap example](https://github.com/mocheng/dissecting-rxjs/blob/master/chapter-08/transform/src/concatMap/drag_drop.html)
  RxJS还有一些非常有意思的特性，包括Scheduler，单元测试等，这些并未在这篇博客中体现，读者可以根据自身需要去了解。
  可以得出，只有在项目中使用RxJS，经过大量实践，才能真正掌握RxJS这套工具。我乖乖地合上了这本书，打算找个项目练练手去了。

## 版本
《深入浅出RxJS》这本书的代码依赖的是RxJs v5.5.0之前的版本，大部分操作符都是采用给Observable类打补丁的形式引用的，我在尝试看这本书的时候，RxJS的版本已经是V6.3.3了，在该版本中，将打补丁的形式完全移除，将所有实例操作符改成了pipeable操作符，其目录存放在"rxjs/operators"，静态操作符直接使用，其目录就在"rxjs/index.js"。