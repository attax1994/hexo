---
title: 你不知道的JavaScript中卷 第二部分
date: 2018-03-19 16:06:19
categories: 
- 读书笔记
tags: 
- books
- 你不知道的JavaScript
---


# 你不知道的JavaScript中卷 第二部分
## 第一章 异步：现在与将来
程序中现在运行的部分和将来运行的部分之间的关系就是异步编程的核心。

### 1.1 分块的程序
最常见的块单位是函数。
从现在到将来的等待，最简单的方法是使用一个通常称为回调函数的函数。

### 1.2 事件循环
一个事件循环处理的示例
```JavaScript
var eventLoop = [];
var event;

var reportError = function(err) {
    console.log('An error happened!');
    console.log(err);
}

// 永远循环
while(true){
    // 一次tick
    if(eventLoop.length) {
        // 移出第一个事件
        event = eventLoop.shift();

        try{
            event();
        }
        catch(err){
            reportError(err);
        }
    }
}
```
诸如setTimeout()等方法，并**不会直接将回调函数挂在事件循环队列中**，而是设置一个定时器，**等定时器到时后，环境会把回调函数放在事件循环中**。

### 1.3 并行线程
并行是关于能够同时发生的事情。
并行最常见的就是进程和线程。进程和线程独立运行，并可能同时运行：多个线程能够共享单个进程的内存。
JS由于其单线程的特性，其不确定性是在函数（事件）顺序级别上，而不是多线程情况下的语句顺序级别。这种函数顺序的不确定性就是常说的竞态条件(race condition)。

### 1.4 并发
#### 1.4.1 非交互
如果进程间没有互相影响的话，不确定性是完全可以接受的。

#### 1.4.2 交互
并发的操作需要相互交流，通过作用域或DOM间接交互。如果出现这样的交互，就需要对它们的交互进行协调以避免竞态的出现。
一个可能的优化是
```JavaScript
// 假设分页情况下
var res = [];

function response(data) {
    res[data.page] = data.items;
}

ajax('url?page=0',response);
ajax('url?page=1',response);
```

#### 1.4.3 协作
并发协作：取到一个长期运行的进程，并将其分割成多个步骤或多批任务，使得其他并发“进程”有机会将自己的运算插入到时间循环队列中交替运行。
本质上是对于一个高性能消耗的操作，将其分割开进行处理。
```JavaScript
var res = [];

function response(data) {
    // 假设1000万条数据，一次处理1000条
    var chunk = data.splice(0, 1000);

    res = res.concat(chuck.map((value)=>{
        return value * 2;
    }));

    // 如果有后续，异步触发
    if(data.length > 0) {
        setTimeout(function() {
            response(data);
        }, 0);
    }
}
```


## 第二章 回调
### 2.2 顺序的大脑
#### 2.2.1 执行与计划
异步不是多任务，而是快速的上下文切换。
编写异步代码的困难之处就在于，这种思考/计划的意识流对我们中的绝大多数来说是不自然的。

### 2.3 信任问题
类型检查/规范化的过程对于函数输入是很常见的，即使是对于理论上完全可以信任的代码。
```JavaScript
function addNumbers(x, y) {
    // 为了防止变成字符串拼接，检查类型，如果不符合就抛出错误
    if(typeof x !== 'number' || typeof y !== 'number') {
        throw new Error('Bad arguments!');
    }

    // 另一种做法是提前转换类型
    x = Number(x);
    y = Number(y);

    return x + y;
}
```

## 第三章 Promise
### 3.1什么是Promise
```JavaScript
var PENDING = 0;
var FULFILLED = 1;
var REJECTED = 2;

function Promise(fn) {
    this._state = PENDING;
    this._value = null;
    // 用树形结构去记录then注册的hanlder，然后等到主Promise决议了，在_excuteThen中操作then返回Promise的状态和结果值
    this._handlers = [];

    if (fn) {
        this._enqueue(fn.bind(null, this._resolve.bind(this), this._reject.bind(this)));
    }
    return this;
}


/**
 * Promise的then方法。
 * 注册新的handler，挂载到handlers树上。
 * 返回该hanlder的promise引用，用于链式调用。
 * 返回的promise的_value取决于then传入的onFulfilled或onRejected函数的返回值。
 * @param {*} onFulfilled 
 * @param {*} onRejected 
 */
Promise.prototype.then = function (onFulfilled, onRejected) {
    var handler = {
        onFulfilled: onFulfilled,
        onRejected: onRejected,
        promise: new Promise(),
    };

    // 先压入堆栈，形成树形结构
    var index = this._handlers.push(handler) - 1;
    // 已经决议了，直接执行then
    if (this._state !== PENDING) {
        this._excuteThen(handler);
    }

    return this._handlers[index].promise;
};


/**
 * Promise的resolve方法。
 * 行为包括状态置位，修改结果，然后执行先前压入堆栈的then
 * @param {*} result 
 */
Promise.prototype._resolve = function (result) {
    // 只有PENDING情况下才能修改状态
    if (this._state === PENDING) {
        this._state = FULFILLED;
        this._value = result;
        this._handlers.forEach(this._excuteThen.bind(this));
    }
};


/**
 * Promise的reject方法
 * @param {*} result 
 */
Promise.prototype._reject = function (result) {
    // 只有PENDING情况下才能修改状态
    if (this._state === PENDING) {
        this._state = REJECTED;
        this._value = reason;
        this._handlers.forEach(this._excuteThen.bind(this));
    }
};


/**
 * 执行then，同时也通知then返回的promise去执行resolve或reject。
 * then中的_value最终使用的值是then中onFulfilled或onRejected的返回值。
 * @param {*} handler 
 */
Promise.prototype._excuteThen = function (handler) {
    if (this._state === FULFILLED && typeof handler.onFulfilled === 'function') {
        // 异步触发then
        this._enqueue(
        handler.promise._resolve.bind(
            handler.promise,
            handler.onFulfilled(this._value)
        )
        );
        // 同步触发then
        /* handler.promise._reject(handler.onFulfilled(this._value)); */
    }
    if (this._state === REJECTED && typeof handler.onRejected === 'function') {
        this._enqueue(
        handler.promise._resolve.bind(
            handler.promise,
            handler.onRejected(this._value)
        )
        );
        /* handler.promise._reject(handler.onRejected(this._value)); */
    }
};


/**
 * 将fn加入事件循环
 * @param {Function} fn 
 */
Promise.prototype._enqueue = function(fn){
    // process.nextTick(fn); // NodeJS
    setTimeout(fn.bind(this), 0);
};
```

### 3.6 Promise模式
#### 3.6.1 Promise.all([...])
Promise.all([...])允许传入一组Promise对象，调用返回的promise会收到一个完成消息，这是一个由所有传入promise的完成消息组成的数组，与指定的顺序一致。
它会在所有成员的promise都完成(fulfilled)后才会完成(fulfilled)。当其中任意一个被拒绝(rejected)，它就立刻进入被拒绝(rejected)状态。

#### 3.6.2 Promise.race([...])
第一个返回的promise为完成，它就会进入完成(fulfilled)状态；第一个返回的promise为失败，它就会进入被拒绝(rejected)状态。

### 3.6.3 其他变体
- none: 类似于all，但是完成和拒绝的情况互换。所有promise都被拒绝，则进入fulfilled状态。
- any：类似于all，但是会忽略拒绝，只需要至少一个完成即可。
- first：类似于any，但是只选取第一个完成的promise。
- last：类似于first，但是只选取最后一个完成的promise。

另一方面，Promise也有原型方法finally，在决议后忽略是否成功，总是会执行。

### 3.8 Promise局限性
#### 3.8.1 顺序错误处理
如果构建了一个没有错误处理函数的Promise链，链中任何地方的任何错误都会在链中一直传播下去。

#### 3.8.2 单一值
Promise只能有一个完成值，或一个拒绝理由。可以进行一定的值封装，但是需要每一步都进行封装和解封。

#### 3.8.3 单决议
Promise只能被决议一次。

#### 3.8.4 惯性
基于Promise对原有的异步回调进行改造依赖于经验，没有通用的实现手段。

#### 3.8.5 无法取消的Promise
单独的Promise不应该可取消，但是取消一个序列（集合在一起的Promise构成的链）是合理的。

#### 3.8.6 性能
相比于不受信任的裸回调，Promise性能上更慢一点。


## 第五章 程序性能
### 5.1 Web Worker
JavaScript目前并没有支持多线程执行的功能。
目前部分浏览器支持通过Web Worker的形式来实现任务并行。通常Web Worker只加载JS文件，浏览器为它启动一个独立的线程，让这个文件在这个线程中为独立的程序运行。
```JavaScript
var w1 = new Worker('http://some.url.com/webworker.js');
```
Worker之间以及它们和主程序之间，不会共享任何作用域或资源，而是通过一个基本的事件消息机制相互联系。比如
```JavaScript
w1.addEventListener('message', function(evt) {
    // evt.data
});
w1.postMessage('something to say');
```
专用Worker和创建它的主程序是一对一的关系，这个message要么来自这个Worker，要么来自主页面。
Worker可以实例化它的子Worker，称为subworker。
要想关闭一个worker，只要在主程序中对Worker调用`terminate()`方法。

#### 5.1.1 Worker环境
在Worker内部是无法访问主程序的任何资源的。
它可以用来执行网络操作（ajax，websocket）以及设置定时器，并可以访问几个重要的全局变量和功能的本地复本，比如navigator、location、JSON和applicationCache。
还可以通过importScript(...)方法向Worker加载额外的JavaScript脚本：
```JavaScript
// Worker内部，该操作是同步的
importScript('foo.js', 'bar.js');
```
Web Worker的应用：
- 处理密集型数学计算
- 大数据集排序
- 数据处理
- 高流量网络通信

#### 5.1.2 数据传递
如果要传递一个对象，可以使用**结构化克隆算法**，把这个对象复制到一边。
或者可以使用**Transferable**对象，发生对象所有权的转移，数据本身不做移动。一旦所有权发生转移，它原来的位置上就会变为null或不可访问，消除多线程编程作用域共享带来的混乱。比如使用postMessage()方法发送一个Transferable对象：
```JavaScript
// 假设foo是一个Uint8Array
postMessage(foo.buffer, [foo.buffer]);
```

#### 5.1.3 共享Worker
使用`SharedWorker`可以创建一整个站点或app的所有页面实例都可以共享的中心Worker。
```JavaScript
var w1 = new SharedWorker('http://some.url.com/sharedWorker.js');
```
SharedWorker使用端口来作为不同程序的唯一标识符，调用程序必须使用Worker的port对象用于通信。
```JavaScript
w1.port.addEventListener('message', handleMessages);
w1.port.postMessage('something');

// 端口连接初始化
w1.port.start();
```
同时在SharedWorker内部，还需要处理额外的`connect`事件，为这个特定的连接提供了端口对象。保持多个连接独立的最简单办法就是使用port上的闭包，把这个连接上的事件侦听和传递定义在`connect`的处理函数内部：
```JavaScript
// 在SharedWorker内部
addEventListener('connect', function(evt) {
    // 这个连接分配的端口
    var prot = evt.ports[0];

    port.addEventListener('message', function(evt) {
        // ...

        port.postMessage('something');

        // ...
    });
    port.start();
});
```

### 5.2 SIMD（单指令多数据）
单指令多数据是一种数据并行(data parallelism)的方式，这不是把程序逻辑分成并行的块，而是并行处理数据的多个位。现代CPU通过数字“向量”（特定类型的数组），以及可以在所有这些数字上并行操作的指令，来提供SIMD功能。这是利用低级指令级并行的底层运算。

### 5.3 asm.js
#### 5.3.1 如何使用asm.js优化
```JavaScript
var a = 42;
var b = a;

// asm优化，确保b是32位整型
var b = a | 0;
```

#### 5.3.2 asm.js模块
对JS性能影响最大的因素是内存分配、垃圾收集和作用域访问。可以使用asm.js模块来解决这些问题。
对于一个asm.js模块来说，需要明确地导入一个严格规范的命名空间——stdlib，以导入必要的符号。同时还需要声明一个堆（heap）并将其传入`var heap = new ArrayBuffer(0x10000); //64k堆`，asm.js就可以在这个缓冲区存储和获取值，不需要付出任何内存分配和垃圾收集的代价。
asm的目标是针对特定的任务处理提供一种优化的方法，比如数学运算和游戏中的图像处理。


## 第六章 性能测试与调优
### 6.1 性能测试
使用Benchmark.js、mocha、karma等测试框架。

### 6.5 微性能
如果某个变量只在一个位置被引用，而别处没有任何引用，那么它的值就会被在线化，即直接用值替换变量。
当递归可以进行展开，引擎就会对其进行采用循环的方式实现。
非关键路径上的优化没有必要，而应该注重可读性。关键路径要注重性能优化。

### 6.6 尾调用优化
使用TCO可以将尾递归转为普通的循环
```JavaScript
function tco(f) {
    var value,
        active = false,
        accumulated = [];

    
    // 在f内部进行递归调用的时候，其实调用的是这个返回的accumulator
    // 这里需要注意的是，要将f中的递归调用函数名使用accumulator被赋予的变量名
    return function accumulator() {
        accumulated.push(arguments);
        if(!active) {
            active = true;
            /**
             * 由于每次f.apply都会导致accumulated被push进一个新的arguments，
             * 所以这个while一直要到全部执行结束才会跳出，
             * 但这种结构只会保持最多两层的调用栈
             */
            while(accumulated.length) {
                value = f.apply(this, accumulated.shift());
            }
            active = false;
            return value;
        }
    }
}

var sum = tco(function(x, y) {
    if (y > 0) {
        // 再次调用的其实是闭包返回的accumulator
        return sum(x + 1, y - 1)
    }
    else {
        return x
    }
});
```
```flow
st=>start: 开始
ed=>end: 结束

cond1=>condition: 检查accumulator.length
(如果是第一次进入，
或者上一次的递归调用中做了push，
这个循环就会继续)
cond2=>condition: 需要进行递归

opm0=>subroutine: 初始化闭包内的变量，
调用闭包返回的函数,
执行accumulated.push(arguments)，
第一次的active判断必然通过
op1-1=>operation: while循环
op1-2=>operation: 取出上次的运行结果，
传参给f.apply
op2-1=>operation: 执行真正的操作
op2-2=>operation: 进入第二层递归调用
op2-3=>operation: 执行
accumulated.push(arguments)，
记录这一次递归调用的结果
op2-4=>operation: 第二层中!active为false，返回
op2-5=>operation: 回到第一层的while循环
op3-1=>operation: 递归调用条件已经不符合，可以返回结果，不再做push
op4-1=>operation: 已经执行完毕，value得到最后的结果

st->opm0->op1-1->cond1
cond1(yes,right)->cond2
cond1(no)->op4-1->ed
cond2(yes,right)->op1-2->op2-1->op2-2->op2-3->op2-4->op2-5(left)->op1-1
cond2(no,right)->op3-1(right)->op1-1
```
