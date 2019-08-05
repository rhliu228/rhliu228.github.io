---
title: 利用Chrome开发者工具探讨运行时性能
date: 2019-08-01 14:57:35
tags:
- Performance
- Javascript
---
[RAIL模型](https://developers.google.cn/web/fundamentals/performance/rail)是一个以用户为中心的性能模型，讲述每个网络应用都会在Response, Animation, Idle, Load四个方面影响用户体验。本文主要讲述如何使用Chrome和DevTools查找影响页面性能的内存问题，包括内存泄漏、内存膨胀和频繁的垃圾回收。
<!-- more -->
* 内存泄漏： 页面随着时间的延长越来越卡顿，这是因为页面中的错误导致页面随着时间的延长使用的内存越来越多
* 内存膨胀： 内存膨胀是指页面为达到最佳速度而使用的内存比本应使用的内存多，这时页面的性能会一直很糟糕
* 频繁垃圾回收： 垃圾回收是指浏览器收回内存，这是由浏览器决定的。当页面出现延迟或者经常暂停时，说明存在频繁垃圾回收，因为回收期间，所有脚本执行都将暂停

以下将从几个方面展开讨论：
* Chrome Performance工具
* 解决性能问题
* 其他工具
## 一. Chrome Performance
平时我们使用chrome开发者工具，用的比较多的工具是Element、Source、Network等，而今天的主角Performance的关注点是运行时性能表现（runtime performance）。
Performance面板主要包括四个部分：
* Controls: 开始记录，停止记录和配置记录期间捕获的信息
* Overview: 页面性能概览
* Main Section主体: 可以查看火焰图，Network活动
* Details: 选择事件后，此窗格会显示与该事件有关的更多信息。 未选择事件时，此窗格会显示选定时间范围的相关信息。

![performance面板](/assets/img/performance/pic1.png)
Figure1: performance面板

### 1.1 Controls

![controls](/assets/img/performance/pic2.png)
Figure2: Controls

#### 1.1.1 记录运行时性能(Record)
按下此按钮，会对当前页面的运行记录进行记录，在记录的过程中，该按钮显示红色。
#### 1.1.2 记录加载性能(Load)
按下后会自动重新加载当前页面并在数秒后停止录制。在生成的火焰图可以看到不同颜色的垂直虚线。蓝线代表DOMContentLoaded事件，绿线代表首次绘制的时间，红线代表load事件。其中**First Paint**标示浏览器渲染任何视觉上不同于导航前屏幕内容的时间点，**First Contentful Paint**表示从输入URL进行导航到浏览器开始渲染DOM第一个字节的时间点，**First Meaningful Paint**表示页面主要内容出现在屏幕的时间，主要内容的定义因页面而异，对于博客来说，它是标题+首评文本，对于搜索引擎，它可能是搜索结果。
![load](/assets/img/performance/pic3.png)
Figure3: 加载性能
#### 1.1.3 清除按钮(Clear)
按下后会清除之前的记录信息，恢复白板状态
#### 1.1.4 录制截屏(Screenshots)
在录制过程中启动截屏，此功能开启后会有一定性能消耗。开启后会在NET下方生成每一帧的截屏
#### 1.1.5 查看内存指标(Memory)：
开启后会在Details面板上方显示一张新的**Memory**图表，并在**NET**图标下方显示**HEAP**图表，HEAP图表与Memory图表的**JS HEAP曲线**展示的均是JS堆内存的占用状况，此外Memory图表还记录了Node节点数、监听函数数目和GPU占用等信息。
#### 1.1.6 垃圾回收(GC)
记录过程中点击后会强制浏览器进行GC垃圾回收活动
#### 1.1.7 捕获设置(Capture)
移动设备的CPU和网络一般比PC端更弱。分析页面时，可以用该组设置模拟移动端设备糟糕的CPU和网速等。
* Disable Javascript Samples
默认情况下，火焰图(见下方)会显示JS函数调用的所有堆栈细节，可以通过启用该选项来禁止这一默认行为。启用选项后，火焰图只会显示JS顶部函数的调用堆栈
* Throttle the network while recording
模拟移动端的网络状态，默认情况下为No throttling
* Throttle the CPU while recording
模拟移动端的CPU架构，默认情况下为No throttling
* Enable advanced paint instrumentation
启用该选项后可以查看浏览器的视图层和详细绘制信息
### 1.2 Overview
![Overview](/assets/img/performance/pic4.png)
Figure4: Overview
#### 1.2.1 FPS（刷新率）
表示一秒之间能够完成多少次重新渲染网页，一次重新渲染代表一帧（frame）。一般的网页动画，需要达到每秒30帧到60帧的频率，才能比较流畅。每秒低于24帧的动画，人眼就能感受到停顿；而如果能达到每秒70帧甚至80帧，就会极其流畅。大多数显示器的刷新频率是**60Hz**，为了与系统一致，以及节省电力，浏览器会自动按照这个频率，与显示器同步刷新（如果可以做到的话），达到最佳的视觉效果。根据公式**T=1/f**，可以得知，每次重新渲染的时间间隔不能超过16.66毫秒，也就是说JavaScript每个任务的耗时，必须少于**16毫秒**。下图的绿色竖线越高，FPS 越高。 FPS 图表上的**红色块**表示FPS很低，很可能存在性能问题。
#### 1.2.2 CPU
此面积图指示消耗CPU资源的事件类型：
  1. 蓝色：网络通信和HTML解析
  2. 黄色：JavaScript执行
  3. 紫色：样式计算和布局，即布局
  4. 绿色：绘制

#### 1.2.3 NET
每个蓝色横杆表示一次网络请求，横杆越长，代表该次请求花费时间越长。

#### 1.2.4 HEAP
内存图标视图，显示堆内存占用状态。随着内存使用量的增长，你会看到图表区域的时间线捕获也会增长。 当图表突然下降时，代表一次垃圾收集器（Garbage collector）运行时的实例，此时清理了引用的内存对象。

### 1.3 Main Section

#### 1.3.1 Flame Graph火焰图
火焰图最初用于展示linux系统CPU调用栈，其y轴表示调用栈，每一层都是一个函数；x 轴表示抽样数，如果一个函数在x轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。
浏览器的火焰图与标准火焰图有两点差异：
1. 它是倒置的（即调用栈最顶端的函数在最下方）
2. x轴是时间轴，而不是抽样次数。

![Flame chart](/assets/img/performance/pic5.png)
Figure5: Flame Chart

上图显示各种事件的信息大集合，上层代表随着时间推移而发生的事件，下面各行是上层事件的子项，由下面的各项组成上层的整体事件。标注有**红色三角形方块**的事件表示该事件存在性能问题。
条的高度与调用堆栈的深度相对应，高调用堆栈只是表示调用了大量的函数。但宽条表示调用表明需要很长时间完成，而这需要优化。

#### 1.3.2 Network 
网络资源请求瀑布图，以不同颜色的条状表示不同的资源
* 黄色：script文件
* 蓝色：html文件
* 紫色： css文件
* 绿色： 媒体文件
* 灰色： 各种其他文件。

![Network chart](/assets/img/performance/pic6.png)
Figure6: Network waterfall

可以点击某个请求条后在Details面板查看该请求的详细信息。左上角存在**深蓝色**方块的请求代表这是一个**高优先级**的请求，意味着这是一个**关键渲染路径**上的请求；反之，一个浅蓝色标识的请求意味着低优先级。

在上图中，对搜狗翻译的html页面请求由三个部分组成： 左侧的实线+中间的长块+右侧的实线。
* 左侧实线标示**Request Sent**前的准备时间
* 右侧实线标示该请求等待主线程的时间
* 而多色长块前面的浅色部分代表**TTFB（Time To First Byte）**（从最初网络请求到从服务器接收到第一个字节的花费时间），深色部分表示资源下载时间（Content Download）。

点击 Chrome Network面板，可以在Timing tab找到对应的请求信息：

![Network timing](/assets/img/performance/pic7.png)
Figure7: Network Timeing

#### 1.3.3 Frames
这是对Overview面板FPS的详细说明，鼠标移到每个帧的色柱上方，会显示该帧耗时和对应的FPS。左右移动鼠标，可以重现当时的屏幕录像。这被称为scrubbing, 可以用来分析动画的各个细节。点击某一帧，可以在Details面板查看该帧的详细信息。

![Frame](/assets/img/performance/pic8.png)
Figure8: Frame

#### 1.3.4 Memory
在Controls点击Memory Checkbox后，会在Details面板上方生成内存时间线（Memory Timeline）， 这里可以看到内存使用按JS堆（与Overview窗格中的HEAP图表相同）、文档、DOM 节点、侦听器和GPU内存细分。
![Memory Timeline](/assets/img/performance/pic9.png)
Figure9: Memory Timeline

### 1.4 Details
Details面板根据用户所选返回范围和事件类型显示对应的具体信息。

1. Summary：所选事件的一个信息汇总

![Summary](/assets/img/performance/pic10.png)
Figure10: Summary Tab

2. Bottom-Up tab：如果要查看哪些活动占据了大部分时间，应使用Bottom-Up tab。该面板根据事件耗时长短，反向列出事件，不过也支持自定义排序和依据分类过滤。

![Bottom-Up tab](/assets/img/performance/pic11.png)
Figure11: Bottom-Up Tab<br/>

其中，**Self Time**表示该事件在所选范围内直接耗费的总时间，不包括任何子事件；**Total Time**表示在该事件或其任何子事件中花费的总时间。

3. Call Tree tab：当你想查看哪些根活动引起的事件占据了大部分时间，应使用Call Tree tab。**根活动是指那些引起浏览器做某些工作的活动**。

![Call Tree tab](/assets/img/performance/pic12.png)
Figure12: Call Tree tab

4. Event Log tab：按照事件在录制过程中的触发顺序一一列举

![Event Log tab](/assets/img/performance/pic13.png)
Figure13: Event Log tab<br/>

下表是一些常见事件属性说明

| 事件 |说明 | 
| :------|:------|
| Parse HTML | Chrome 执行其 HTML 解析算法。 |
| XHR Ready State Change | XMLHTTPRequest 的就绪状态已发生变化 | 
| Timer Fired | 使用 setInterval() 或 setTimeout() 创建的定时器已被触发。 |
| Function Call | 已调用一个函数|
| Recalculate style | Chrome 重新计算了元素样式|
| GC Event | 发生垃圾回收|
| DOMContentLoaded | 浏览器触发[DOMContentLoaded](https://developer.mozilla.org/en-US/docs/Web/API/Document/DOMContentLoaded_event)事件。当页面的所有 DOM 内容都已加载和解析时，将触发此事件|

更多事件说明可参考[
Timeline Event Reference](https://developers.google.cn/web/tools/chrome-devtools/evaluate-performance/performance-reference)

### 1.5 其他
浏览器除了负责网络请求工作和执行脚本以外，当用户在浏览器输入一个网页后，浏览器还会在背后做很多工作。

![WebKit main flow](/assets/img/performance/pic14.png)
Figure14: WebKit main flow<br/>

利用performance工具，可以在浏览器中查看关于浏览器布局和绘制的各个活动的详细信息。
#### 1.5.1 查看layers
1. 在controls的capture中勾选Enable advanced paint instrumentation checbox
2. 点击绘制并生成图表
3. 在Frame面板中选择某一帧，这时可以看到在Details面板中多了一个新的Tab：**Layers**。

![layer](/assets/img/performance/pic15.png)
Figure15: layer<br/>

#### 1.5.2 paint profiler
1. 在controls的capture中勾选Enable advanced paint instrumentation checbox
2. 点击绘制并生成图表
3. 在火焰图中点击选择一个paint事件，这时可以看到在Details面板中多了一个新的Tab：**Paint Profiler**。

![paint profiler](/assets/img/performance/pic16.png)
Figure16: paint profiler<br/>

#### 1.5.3 render和layout
1. 打开Command Menu, 输入Rendering，调出Rendering面板
2. 选中Paint Flashing checkbox

## 二. 解决性能问题

### 2.1 Chrome 任务管理器实时监视内存使用
1. Chrome 主菜单并选择 More tools > Task manager，打开任务管理器
2. 右键点击任务管理器的表格标题并启用JavaScript memory

![Task Manager](/assets/img/performance/pic17.png)
Figure17: Task Manager<br/>

* Memory 列表示**原生内存(native memory)**，DOM节点存储在原生内存中。 如果此值正在增大，则说明正在创建DOM节点。
* JavaScript Memory列表示JS堆。此列包含两个值。实时数字表示页面上的可到达对象正在使用的内存量。如果此数字在增大，要么是正在创建新对象，要么是现有对象正在增长。


 > 渲染器内存（Renderer memory）是渲染页面的进程的内存总和：原生内存 + 页面的JS堆内存 + 页面启动的所有web worker的JS堆内存。原生对象是JavaScript堆之外的任何对象，存放于原生内存中。 


### 2.2 使用Memory Timeline可视化内存泄漏
![Memory Timeline2](/assets/img/performance/pic18.png)
Figure18: Memory Timeline<br/>
如果JS堆大小或节点大小不断增大，则可能存在内存泄漏。

### 2.3 使用Heap snapshot查看DOM外引用的内存泄露
Heap snapshot可以按页面的JavaScript对象和相关DOM节点显示内存分配。快照有三种视图，可以从不同角度查看快照。
1. 打开Chrome Memory面板，点击Heap snapshot
2. 点击Take snapshsot，等待生成快照

#### 2.3.1 summary可以显示按构造函数名称分组的对象。

![Memory summary](/assets/img/performance/pic19.png)
Figure19: Memory summary<br/>

表格标题的字段含义分别是： 
1. Constructor 表示使用此构造函数创建的所有对象，后面跟着对象实例数。
2. [Shallow Size](https://developers.google.cn/web/tools/chrome-devtools/memory-problems/memory-101)列显示通过特定构造函数创建的所有对象浅层大小的总和。浅层大小是指对象自身占用的内存大小。
3. Retained Size 列显示同一组对象中最大的保留大小。将对象本身连同其无法从GC根到达的相关对象一起删除后释放的内存大小，称为[保留大小(Retained Size)](https://developers.google.cn/web/tools/chrome-devtools/memory-problems/memory-101)。
4. Distance 显示该节点距根节点的最短路径距离。

#### 2.3.2 Containment视图采用自顶向下的方式显示对象的结构

![Memory Containment](/assets/img/performance/pic20.png)
Figure20: Memory Containment<br/>

#### 2.3.3 Statistics视图按照数据类型划分内存的使用

![Memory Statistics](/assets/img/performance/pic21.png)
Figure21: Memory Statistics<br/>


#### 2.3.4 Memory Summary视图跟踪DOM泄漏。
只有页面的DOM树或JavaScript码不再引用DOM节点时，DOM节点才会被作为垃圾进行回收。如果某个节点已从DOM树移除，但某些JavaScript仍然引用它，我们称此节点为“**detached**”。已分离的DOM节点是内存泄漏的常见原因。可以使用Heap snapsho找出已分离的DOM节点并移除它们。

1. 切换到Summary视图
3. 在搜索框里输入Detached，过滤出Detached Dom节点


![Heap snapshot](/assets/img/performance/pic22.png)
Figure22: Heap snapshot<br/>

### 2.4 使用Allocation timeline查看JS堆内存泄露
1. 打开Chrome Memory面板，点击Allocation instrumentation on timeline
2. 点击Start，等待生成快照
3. 执行怀疑存在内存泄漏的部分，然后点击Stop

![Allocation timeline](/assets/img/performance/pic23.png)
Figure23: Allocation timeline<br/>

Allocation timeline显示新分配的堆内存并标示对应的保留路径（retaining path）。每个条形的高度对应于最近分配的对象的大小，条形的颜色指示这些对象是否仍然存在于最终堆快照中。 **蓝色**条表示在录制结束时仍然存在内存的对象，**灰色**条表示在录制期间分配的堆对象，但此后已被垃圾收集。若一个蓝色条在录制结束后没有按照期望的那样被GC收集，可以拖动时间轴的滑块定位到特定的时间范围，并在下方查看对象的保留路径来查看为什么没有被GC回收。
### 2.5 使用Allocation Sampling查看分配给函数的内存
1. 打开Chrome Memory面板，点击Allocation Sampling
2. 点击Start，与页面交互
3. 点击Stop，等待生成Profiler

![Allocation Sampling](/assets/img/performance/pic24.png)
Figure24: Allocation Sampling<br/>

### 2.6 使用Javascript Profiler识别CPU开销大的函数
使用CPU分析器准确地记录调用了哪些函数和每个函数花费的时间：
1. 打开Chrome Javascript Profiler面板，点击Record Javascript CPU Proile
2. 点击Start，与页面交互
3. 点击Stop，等待生成Profiler

默认会生成火焰图： 

![Javascript Profiler1](/assets/img/performance/pic25.png)
Figure25: Javascript Profiler1<br/>

点击Heavy (Bottom Up)，将按照函数对性能的影响列出函数，可以检查函数的调用路径：

![Javascript Profiler2](/assets/img/performance/pic26.png)
Figure26: Javascript Profiler2<br/>

### 2.7 发现频繁的垃圾回收
如果感觉页面经常暂停，则可能存在垃圾回收问题。
JavaScript的内存模型建立在称为垃圾收集器的技术之上。在许多语言中，程序员直接负责从内存的堆中分配和释放内存。但是，垃圾收集器系统代表程序员管理此任务，这意味着当程序员解除引用时，对象不会直接从内存中释放，而是将决定权交给GC。GC需要对活动和非活动对象执行一些统计分析，这需要一段时间来执行，回收期间，所有脚本执行都将暂停。除此之外，系统本身决定何时运行GC，程序员无法控制此操作，在代码执行期间的任何时候都可能发生GC回收活动。经常创建/释放对象的过程称为“**memory churn**”，因为这意味着程序不得不经常阻塞以等待GC回收完成。应避免以下情形出现：
 
![Bad GC](/assets/img/performance/pic27.png)
Figure27: Bad GC<br/>

可以通过performance overview面板的HEAP或者Memory Timeline来观察GC的活动，JS堆大小频繁上升和下降指示存在频繁的垃圾回收，这时页面会存在卡顿现象。

## 三. 更多
1. 可以使用Chrome的Audit面板来测试和分析网站，测试完毕后LightHouse会给出性能评估和优化建议
2. 使用[Performance API](https://w3c.github.io/navigation-timing/#introduction)来监控系统性能并进行数据上报分析

## 参考：
* [Static Memory Javascript with Object Pools](https://www.html5rocks.com/en/tutorials/speed/static-mem-pools/?redirect_from_locale=zh)
* [Fix Memory Problems](https://developers.google.cn/web/tools/chrome-devtools/memory-problems/)
* [如何读懂火焰图？](http://www.ruanyifeng.com/blog/2017/09/flame-graph.html)
* [Performance Analysis Reference](https://developers.google.cn/web/tools/chrome-devtools/evaluate-performance/reference#rendering)
* [How to Record Heap Snapshots](https://developers.google.cn/web/tools/chrome-devtools/memory-problems/heap-snapshots)
* [Navigation Timing Level 2](https://w3c.github.io/navigation-timing/#introduction)