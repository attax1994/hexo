---
title: Vue源码解析13--core.observer-观察者
date: 2019-04-26 18:31:13
categories: 
- 前端开发
tags: 
- 技术原理
---

## Observer观察者

Vue要定义一个响应式的属性，本质上是改造其getter/setter。

对于getter，具体的操作是：

1. 执行原本的getter（如果用户做了定义），获得value。
2. 如果是响应式的属性，执行依赖收集，即将该`dep`加入`watcher.deps`中，将`watcher`加入`dep.subs`中，交叉引用（当然也要做查重，避免加入重复的引用）。
3. 深度（deep）模式下，它的成员也有observer，执行它对应的依赖收集。成员为Array的情况下，遍历它，并执行依赖收集。
4. 返回 value。



对于setter，具体的操作是：

1. 先触发getter，获得上一次的值，并收集依赖。
2. 对比当前值和上一次的值，如果没变化，直接返回。
3. 如果发生了变化，执行原本的setter（如果用户做了定义），完成赋值操作。
4. 深度（deep）模式下，尝试为成员创建Observer。
5. **操作它的`dep`执行`notify()`方法，进而触发该`dep`下所有`watcher`的`update()`方法**。



对于一个vm的更新，就可以描述为（假设修改了`$data`中的某个属性值）：

1. 触发该属性改造后的setter。
2. setter中触发getter进行依赖收集，确认传入的值和之前不同，先执行原来定义的setter，然后触发这个属性的`dep`（在`defineReactive()`的闭包中创建）发出`notify()`更新通知。
3. 根据该`dep`中`dep.subs`的记录，触发所有订阅这个`dep`的`watcher`的`update`方法。
4. 在`watcher.update()`中判断执行类型，通常`$data`属性改变触发的是由`scheduler`负责的`queueWatcher(this)`异步更新。
5. `scheduler`将这个`watcher`加入更新队列，`nextTick()`中flush整个队列，调用`watcher.run()`来执行更新。
6. `watcher.run()` 中执行该`watcher`的`watcher.get()`方法。
7. `watcher.get()`中，执行`this.getter()`，即在`lifecycle`中`mountComponent()`在创建这个Watcher实例时，传入的`updateComponent()`方法，触发`vm._update(vm._render(), hydrating)`。
8. `vm._render()`负责生成新的`vnode`，`vm._update()`对比新`vnode`和原`vnode`，执行`patch`算法，并将其挂载到组件的实例上去，更新完成。



### 建模

对于整个Observer模块，共有四个核心概念

1. `Observer`类，附着到每个被观察的对象，作为其`__ob__`属性，负责改造其属性为`getter/setter`形式。`dep`属性为其`Dep`，用于收集依赖并发出更新通知。
2. `Dep`类，负责依赖收集和发布更新通知，可以让多个watcher去订阅它。它记录了所有订阅它的watcher，放在`subs`中。其静态属性`Dep.target`用于指示当前正在执行的watcher。
3. `Watcher`类，解析getter表达式，收集依赖，并且在getter所指向的值发生改变时触发回调。`computedWatcher`放在`vm._computedWatchers`中，其他watcher放在`vm._watchers`中，vm中负责render的watcher为`vm._watcher`。
4. `Scheduler`调度器，用于记录当前tick下需要进行异步更新的watcher队列，调度这些watcher的异步更新，在`nexttick()`中执行。执行完成后，触发对应vm的`activated`和`updated`钩子，并清理现场。

