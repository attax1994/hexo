---
title: （译）任务（Tasks），微任务（microtasks），队列（queues）与调度（schedules）
date: 2018-05-11 15:50:36
categories: 
- 前端开发
tags:
- 技术原理
---

原文地址：[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules)

注：本文非原文的通篇翻译，而是节选部分深入原理的部分进行翻译。

## 任务（Tasks）、微任务（microtasks）、队列（queues）与任务安排（schedules）

试试下面这一段JavaScript

```JS
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

这些日志的输出应当是什么顺序呢？



## 试一试

```JS
script start
script end
promise1
promise2
setTimeout
```

正确答案是：`script start`，`script end`，`promise`，`promise2`，`setTimeout`，但是考虑到各个浏览器支持方面，它就毫无规律可言了。

Edge、Firefox 40、iOS Safari 和 desktop Safari 8.0.8，这些浏览器会先输出`setTimeout`，然后才是`promise1`和`promise2`，虽然这应当是个竞态问题。比较奇怪的是，Firefox 39 和 Safari 8.0.7却一直能输出正确结果。



## 发生了什么？

要理解这个这个情况，就需要先理解事件循环（下称Event Loop）如何处理任务（下称tasks）与微任务（下称microtasks）。这种情况可能在你第一次碰到的时候把你绕头晕，所以在继续深入之前，请做好心理准备……

每个“线程”都有它自己的event loop，也就是说每个web worker也都有它们各自的event loop，这使得它能够独立地执行，因此所有在同一源站点（origin）上的所有窗口共用一个event loop，这让他们能够互相同步地沟通起来。event loop一遍又一遍地运行着，执行队列中的任务。每个event loop都有多种任务提供者（source），它们保证了任务在这些source中的执行顺序（像IndexedDB这样的规范会定义自身的执行规则），但是浏览器会决定在每轮循环中，选取哪个source，从而执行它内部定义的tasks。这使得浏览器更偏向于首先执行与性能相关的任务，比如处理用户输入。

浏览器将**tasks**进行安排，从而它就能通过内部作用域去访问 JS/DOM 层面，并且保证这些操作按顺序执行。在task与task之间，浏览器会渲染更新的状态。从鼠标点击事件到执行对应的事件回调，就涉及对任务进行安排。解析HTML与上例中的输出`setTimeout`也是一个运作原理。

`setTimeout`等待一个给定的延时，然后为它的回调函数安排一个新任务。这也是为什么`setTimeout`在`script end`之后输出——因为输出`script out`是第一个任务中的一部分，并且`setTimeout`在一个单独的任务中输出。



Microtasks通常被安排在当前运行栈结束之后立即执行，比如反应一系列的操作，或是让一些异步的操作在无需创建新任务的情况下执行。只要没有其他 JS 语句处于运行中的状态，microtask队列就会在注册它的回调函数执行完后立刻执行，这同时也意味着，它在每个任务的最后阶段执行。其他在microtask中注册的microtasks会被添加到队列尾部并执行。microtasks包括MutationObserver的回调函数，还有上例中的Promise回调。

一旦Promise发生决议，或者已经决议了，它就会将对应回调函数作为一个microtask加入队列。这保证了Promise的回调在Promise已经决议的情况下，也是异步的执行的。所以在一个决议的Promise上调用`.then(yey, nay)`，会立刻将一个microtask加入队列。这也是为什么`promise1`和`promise2`在`script end`之后输出——因为当前运行栈必须在microtasks处理之前结束。而`promise1`和`promise2`在`setTimeout`之前输出，是因为microtasks总会在下一个task之前运行。



## 如何判断某个异步操作使用microtask还是task

一种方法是自行测试。查看它们的日志输出是近似于promise还是setTimeout。当然你得注意浏览器实现的方式是否符合标准。

比较恰当的方式是查规范。比如，[step 14 of `setTimeout`](https://html.spec.whatwg.org/multipage/webappapis.html#timer-initialisation-steps) 将task加入队列，而 [step 5 of queuing a mutation record](https://dom.spec.whatwg.org/#queue-a-mutation-record) 将microtask加入队列。

正如上述所言，在ECMAScript领域，microtasks被称为`jobs`，调用 [step 8.a of `PerformPromiseThen`](http://www.ecma-international.org/ecma-262/6.0/#sec-performpromisethen)  的`EnqueueJob`可以将microtask加入队列。



## 第一个例子

假设HTML为

```html
<div class="outer">
  <div class="inner"></div>
</div>
```

给定下面的JS代码，当我点击`div.inner`时候会输出什么？

```js
// 得到这两个div的引用
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// MutationObserver监听外层div的变化
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// click回调
function onClick() {
  console.log('click');

  setTimeout(function() {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

// 给这两个元素都注册click事件
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```

正确的输出顺序为：

```JS
// div.inner的click回调
click
// 回调后先注册Promise.then()的microtask，后触发DOM更新，MutationObserver注册一个microtask
promise
mutate

// 事件冒泡到div.outer
click
promise
mutate

// 两个setTimeout分别为新的task
timeout
timeout
```

需要注意的是，如果使用`inner.click()`来触发点击，就会使两个`click`回调被同步注册，结果就会变成：

```JS
// 同步注册两个click回调
click
click
// 注册第一个的microtask
promise
mutate

// MutationObserver在当前task中已经有该类型microtask的情况下，不触发第二个回调
promise

// 两个setTimeout分别为新的task
timeout
timeout
```



##  它们重要吗？

是的，它会在暗中坑了你。我在试着用Promise，而不是奇葩的`IDBRequest`对象， 给IndexedDB做个封装库的时候，碰到了这个问题。它让IDB变得很有意思。

当IDB触发一个success事件，相关的处理对象在`dispatch(step 4)`后不能操作。如果我创建一个Promise，在这个success事件触发的时候resolve，那么回调函数应当在`step 4`之前运行，那时处理对象还是可用的。不过它们在chrome以外的浏览器中就不这么运行了，也就使得这个库失效了。

你可以在Firefox中绕过这个问题，因为Promise的兼容库，比如 [es6-promise](https://github.com/jakearchibald/es6-promise) 使用MutationObserver来执行回调，确实使用了microtask。Safari看起来即使有了兼容措施，也会受制于竞态问题，但那可能只是因为他们对IDB糟糕的实现方式。不幸的是，在IE/Edge下，他们必然会有问题，因为mutation事件并不是在回调之后处理。



## 你学到了！

总体来说：

- Tasks按顺序运行，并且浏览器会在它们之间做渲染操作。
- Microtasks按顺序运行，并且执行方式为：
- - 只要没有其他 JS 语句在运行，就会在每个回调后运行
  - 在每个task的最后运行

