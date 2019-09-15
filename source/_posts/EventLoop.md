---
title: 比较Node和浏览器的Event Loop
date: 2019-09-14 20:52:00
tags: 
- Javascript
- Node
---
一直以为大致搞懂浏览器的eventLoop就差不多了，直到我看了《Node.js调试指南》后，才发现把浏览器和Node环境的eventLoop混在一起谈是不合适的。虽然它们的任务源确实存在很大区别，但更重要的是，Node下eventloop的“读取任务队列”划分为6个阶段，从而导致两者在优先级队列的执行次序上有很大不同。 这篇文章主要阐述浏览器和node环境的eventloop异同。
<!-- more -->
JavaScript是单线程运行，所以异步操作特别重要。为了管理异步回调， Javascript采取事件循环（Event Loop）的策略：
* 所有任务都在主线程上运行，形成一个执行栈（Execution Context Stack）
* 在主线程之外还存在一个或者多个“任务队列”（Task Queue），系统把异步任务放到“任务队列”中。
* 一旦“执行栈”中的任务执行完毕，系统就会读取“任务队列”。某个异步任务结束等待状态，从“任务队列”进入执行栈，得以执行。
* 主线程不断重复第三步。

### 1. 浏览器与Event Loop
[HTML规范](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop)中对Event Loop的定义如下：
> To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. Each agent has an associated event loop.

同时规定：
* An event loop has one or more task queues. A task queue is a set of tasks.（一个event loop可以有1个或多个task queue）。
* Per its source field, each task is defined as coming from a specific task source. For each event loop, every task source must be associated with a specific task queue.（每个task定义时都有一个task source，从同一个task source来的task必须放到同一个task queue，从不同源来的则被添加到不同的task queue）。
* Tasks encapsulate algorithms that are responsible for such work as: Events,Parsing,Callbacks,Using a resource,Reacting to DOM manipulation,etc.


每个(task source对应的)task queue都保证自己队列的先进先出的执行顺序，但event loop的每个turn，是由浏览器决定从哪个task source挑选task。这允许浏览器为不同的task source设置不同的优先级，比如为用户交互设置更高优先级来使用户感觉流畅。

#### 1.1 macrotask
规范里提及的task又称macrotask，从[webappapis.html#generic-task-sources](https://html.spec.whatwg.org/multipage/webappapis.html#generic-task-sources)可以看出，
task源包括：
* DOM 操作任务源：如元素以非阻塞方式插入文档
* 用户交互任务源：如鼠标键盘事件。用户输入事件（如 click） 必须使用 task 队列
* 网络任务源：如 XHR 回调
* history 回溯任务源：使用 history.back() 或者类似 API

此外 setTimeout、setInterval、IndexDB 数据库操作等也是任务源。总结来说，常见的 task 任务有：

* 事件回调
* XHR 回调
* IndexDB 数据库操作等 I/O
* setTimeout / setInterval
* history.back

#### 1.2 microtask
从[webappapis.html#microtask](https://html.spec.whatwg.org/multipage/webappapis.html#microtask)可以知道，每一个 eventloop 都有一个 microtask 队列。microtask 会放在 microtask 队列而非 task 队列中。
一般来说，microtask 包括：
* Promise.then
* MutationObserver
* Object.observe

#### 1.3 Processing model
规范[webappapis.html#event-loop-processing-model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)描述了eventloop的处理过程，简述为以下步骤：
1.  从 task 队列（一个或多个）中选出最老的一个 task，执行它。
2. 执行 microtask 检查点（checkpoint），执行microtask队列中的所有 microtask，直到队列为空。如果microtask又添加了新的 microtask，直接放进本队列末尾。
3. 执行UI render（可选）。
 * 首先判断document在此时间点渲染是否会“获益”。浏览器只需保证 60Hz 的刷新率即可（在机器负荷重时还会降低刷新率），若eventloop频率过高，即使渲染了浏览器也无法及时展示。**所以并不是每轮 eventloop 都会执行 UI Render**。
 * 执行各种渲染所需工作，如 触发 resize、scroll 事件、建立媒体查询、运行 CSS 动画等等
 * **执行animation frame callbacks**
 * **执行IntersectionObserver callback**
 * 渲染 UI

[promise A+规范](https://promisesaplus.com/)里规定promise不能返回自己，因为promise.then会把回调注册到本次microtask队列中，从而导致第二步中的microtask队列永远不可能为空，造成**死循环**。


#### 1.4 microtask 的执行时机
 观察以下代码：
 ```
setTimeout(() => {
  console.log('A')
}, 0)
requestAnimationFrame(() => {
  console.log('B')
  Promise.resolve().then(() => {
    console.log('C')
  })
})
 ```
 打印结果为："B C A"或者"A B C"（B，C一定紧挨着输出）。从中可以知道，requestAnimationFrame 中注册的 microtask 并没有在下一轮 eventloop 的 task 之后执行，而是直接在本轮 eventloop 中紧跟着 requestAnimationFrame 执行了。也就是说一轮 eventloop 中有可能执行多次 microtask

### 2. Node与Event Loop
 [Node官网](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)对Event Loop的描述如下：
 > The event loop is what allows Node.js to perform non-blocking I/O operations — despite the fact that JavaScript is single-threaded — by offloading operations to the system kernel whenever possible.

 Node.js采用V8作为js的解析引擎，而I/O处理方面使用了libuv，libuv是一个基于事件驱动的跨平台抽象层，封装了不同操作系统一些底层特性，对外提供统一的API，事件循环机制也是它的内部实现。

 Event Loop的“读取任务队列”有6个阶段（phase）：
 ```
    ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
 ```

* event loop 的每个phase都有一个FIFO任务队列
* 当 event loop 到达某个phase时，将执行该阶段的任务队列，直到队列清空或执行的回调达到系统上限后，才会转入下一个阶段
* 当所有phase被顺序执行一次后，称 event loop 完成了一个 tick

每个phase的作用如下：
* **timers**： 执行setTimeout()和setInterval()中到期的callback
* **pending callbacks**：上一轮循环中有少数的I/O callbacks会被延迟到这一轮的这一阶段执行
* **idle，prepare**：仅内部使用
* **poll**：执行I/O callback，在适当的条件下，node会阻塞在这个阶段
* **check**：执行setImmediate()的callback
* **close callbacks**：执行close事件的callback，例如socket.on('close', func)

#### 2.1 poll阶段
poll阶段主要有两个功能，如下所述。
* 当timers的定时器到期后，执行定时器（setTimeout和setInterval）的callback
* 执行poll队列里的I/O callback

如果Event Loop进入了poll阶段，且代码未设定timer，则可能发生以下情况。
* 如果poll queue不为空，则event loop将同步queue里的callback，直至queue为空，或者执行的callback达到系统上限
* 如果poll queue为空，则可能发生以下情况：
  * 如果代码用setImmediate()设定了callback，则event loop将结束poll阶段并进入check阶段，执行check阶段的queue
  * 如果代码没有使用setImmediate()，则event loop将阻塞在这个阶段，等待callback进入poll queue，如果有callback进来则立即执行

一旦poll queue为空，则Event Loop将检查timers，如果有timers的时间到期，则Event Loop将回到timers阶段，然后执行timer queue

#### 2.2 process.nextTick()
process.nextTick()不在Event Loop的任何阶段执行，而是在各个阶段切换的中间执行，即从一个阶段切换到下一个阶段前执行。node把macroTask定义为在每个阶段执行的任务，而microtask则是指在每个阶段之间执行的任务。即上述6个阶段都是macrotask，而process.nextTick属于microtask。
```
const promise = Promise.resolve();
promise.then(() => {
    console.log('promise');
});
process.nextTick(() => {
    console.log('nextTick');
});

// nextTick
// promise
```

二者优先级不一样，process.nextTick的microtask queue总是优先于promise的microtask执行。

### 3. Node.js 与浏览器的 Event Loop 差异
浏览器环境下，microtask的任务队列是每个macrotask执行完之后执行。

![eventloop in browser](/assets/img/eventloop/browser.jpg)

在Node.js中，microtask会在事件循环的各个阶段之间执行，也就是一个阶段执行完毕，才会去执行microtask队列的任务。

![eventloop in nodeJs](/assets/img/eventloop/node.jpg)

### 4. 实例
观察以下代码：
```
setTimeout(()=>{
    console.log('timer1')

    Promise.resolve().then(function() {
        console.log('promise1')
    })
}, 0)

setTimeout(()=>{
    console.log('timer2')

    Promise.resolve().then(function() {
        console.log('promise2')
    })
}, 0)

//浏览器输出
timer1
promise1
timer2
promise2

// node输出（node版本v8.11）
timer1
timer2
promise1
promise2
```

在浏览中，两个timer被当作两个macrotask，所以先输出timer1, promise1，再输出timer2，promise2。在node环境下，timer1和timer2都处于timers阶段，首先执行timer1的回调函数，并将promise1.then放入microtask队列，此时队列仍不为空，接着执行timer2，将promise2.then放入microtask队列。
timers阶段结束后，执行microtask队列的所有回调，打印promise1，promise2。（node11之后，node为了保持和浏览器的一致，一次只执行一个timers回调）


参考： 
* [深入探究 eventloop 与浏览器渲染的时序问题](https://github.com/jin5354/404forest/issues/61)
* [event-loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop)
* [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
* [promise A+](https://promisesaplus.com/)