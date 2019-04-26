---
title: Vue源码解析15--core.observer.dep-Dep类
date: 2019-04-26 18:31:15
categories: 
- 前端开发
tags: 
- 技术原理
---

## Dep类

`Dep`类，负责依赖收集和发布更新通知，可以让多个watcher去订阅它。它记录了所有订阅它的watcher，放在`subs`中。其静态属性`Dep.target`用于指示当前正在执行的watcher。

```typescript
let uid = 0

/**
 * dep是一个可以让多个指令去订阅它的观察者
 */
export default class Dep {
  static target: ?Watcher
  id: number
  subs: Array<Watcher>

  constructor() {
    this.id = uid++
    this.subs = []
  }

  // 添加一个Watcher
  addSub(sub: Watcher) {
    this.subs.push(sub)
  }

  // 移除一个Watcher
  removeSub(sub: Watcher) {
    remove(this.subs, sub)
  }

  // 依赖收集，当存在Dep.target的时候添加观察者对象
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  // 通知所有订阅者，发生了update
  notify() {
    // 保证subs为Array
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

/**
 * Dep.target指示当前正在做检查的watcher。
 * 它是一个静态属性，在全局是独一无二的，因为同一时间只会有一个watcher在做检查
 */
Dep.target = null
const targetStack = []

// push一个Watcher进入targetStack，Dep.target记录最新的一条记录
export function pushTarget(_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

// pop顶层Watcher退出targetStack，提供给Dep.target
export function popTarget() {
  Dep.target = targetStack.pop()
}
```

