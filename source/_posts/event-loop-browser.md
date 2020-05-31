---
title: event-loop-browser
date: 2020-05-31 18:28:39
tags: event-loop,browser
---

### 思考的问题：
- 单线程如何做到异步
- 事件循环的过程是怎样的
- `macrotask` 和 `microtask` 是什么，它们有何区别

### 单线程和异步
提到`js`，就会想到`单线程`，`异步`，那么单线程是如何做到异步的呢？概念先行，先要了解下单线程和异步之间的关系。

`js`的任务分为 `同步` 和 `异步` 两种，它们的处理方式也不同，同步任务是直接在主线程上排队执行，异步任务则会被放到任务队列中，若有多个任务（异步任务）则要在任务队列中排队等待，任务队列类似一个缓冲区，任务下一步会被移到调用栈（`call stack`），然后主线程执行调用栈的任务。

单线程是指`js`引擎中负责解析执行`js`代码的线程只有一个（主线程），即每次只能做一件事，而我们知道一个`ajax`请求，主线程在等待它响应的同时是会去做其它事的，浏览器先在事件表注册`ajax`的回调函数，响应回来后回调函数被添加到任务队列中等待执行，不会造成线程阻塞，所以说`js`处理`ajax`请求的方式是异步的。

总而言之，检查调用栈是否为空，以及确定把哪个`task`加入调用栈的这个过程就是事件循环，而`js`实现异步的核心就是事件循环。

### 调用栈和任务队列
顾名思义，调用栈是一个栈结构，函数调用会形成一个栈帧，帧中包含了当前执行函数的参数和局部变量等上下文信息，函数执行完后，它的执行上下文会从栈中弹出。

下图就是调用栈和任务队列的关系图



事件循环
关于事件循环，HTML规范的介绍

There must be at least one event loop per user agent, and at most one event loop per unit of related similar-origin browsing contexts.
An event loop has one or more task queues.
Each task is defined as coming from a specific task source.

从规范理解，浏览器至少有一个事件循环，一个事件循环至少有一个任务队列（macrotask），每个外任务都有自己的分组，浏览器会为不同的任务组设置优先级。

macrotask & microtask
规范有提到两个概念，但没有详细介绍，查阅一些资料大概可总结如下：

macrotask：包含执行整体的js代码，事件回调，XHR回调，定时器（setTimeout/setInterval/setImmediate），IO操作，UI render

microtask：更新应用程序状态的任务，包括promise回调，MutationObserver，process.nextTick，Object.observe

其中setImmediate和process.nextTick是nodejs的实现，在nodejs篇会详细介绍。

事件处理过程
关于macrotask和microtask的理解，光这样看会有些晦涩难懂，结合事件循坏的机制理解清晰很多，下面这张图可以说是介绍得非常清楚了。



总结起来，一次事件循环的步骤包括：

1. 检查macrotask队列是否为空，非空则到2，为空则到3
2. 执行macrotask中的一个任务
3. 继续检查microtask队列是否为空，若有则到4，否则到5
4. 取出microtask中的任务执行，执行完成返回到步骤3
5. 执行视图更新
mactotask & microtask的执行顺序


读完这么多干巴巴的概念介绍，还不如看一段代码感受下

console.log('start')

setTimeout(function() {
  console.log('setTimeout')
}, 0)

Promise.resolve().then(function() {
  console.log('promise1')
}).then(function() {
  console.log('promise2')
})

console.log('end')
打印台输出的log顺序是什么？结合上述的步骤分析，系不系so easy~

首先，全局代码（main()）压入调用栈执行，打印start；

接下来setTimeout压入macrotask队列，promise.then回调放入microtask队列，最后执行console.log(‘end’)，打印出end；

至此，调用栈中的代码被执行完成，回顾macrotask的定义，我们知道全局代码属于macrotask，macrotask执行完，那接下来就是执行microtask队列的任务了，执行promise回调打印promise1；

promise回调函数默认返回undefined，promise状态变为fullfill触发接下来的then回调，继续压入microtask队列，event loop会把当前的microtask队列一直执行完，此时执行第二个promise.then回调打印出promise2；

这时microtask队列已经为空，从上面的流程图可以知道，接下来主线程会去做一些UI渲染工作（不一定会做），然后开始下一轮event loop，执行setTimeout的回调，打印出setTimeout；

这个过程会不断重复，也就是所谓的事件循环。

视图渲染的时机
回顾上面的事件循环示意图，update rendering（视图渲染）发生在本轮事件循环的microtask队列被执行完之后，也就是说执行任务的耗时会影响视图渲染的时机。通常浏览器以每秒60帧（60fps）的速率刷新页面，据说这个帧率最适合人眼交互，大概16.7ms渲染一帧，所以如果要让用户觉得顺畅，单个macrotask及它相关的所有microtask最好能在16.7ms内完成。

但也不是每轮事件循环都会执行视图更新，浏览器有自己的优化策略，例如把几次的视图更新累积到一起重绘，重绘之前会通知requestAnimationFrame执行回调函数，也就是说requestAnimationFrame回调的执行时机是在一次或多次事件循环的UI render阶段。

以下代码可以验证

setTimeout(function() {console.log('timer1')}, 0)

requestAnimationFrame(function(){
	console.log('requestAnimationFrame')
})

setTimeout(function() {console.log('timer2')}, 0)

new Promise(function executor(resolve) {
	console.log('promise 1')
	resolve()
	console.log('promise 2')
}).then(function() {
	console.log('promise then')
})

console.log('end')
运行结果截图如下

运行结果1运行结果2

可以看到，结果1中requestAnimationFrame()是在一次事件循环后执行，而在结果2，它的执行则是在三次事件循环结束后。

总结
事件循环是js实现异步的核心
每轮事件循环分为3个步骤：
a) 执行macrotask队列的一个任务
b) 执行完当前microtask队列的所有任务
c) UI render

浏览器只保证requestAnimationFrame的回调在重绘之前执行，没有确定的时间，何时重绘由浏览器决定