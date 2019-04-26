---
title: Vue源码解析18--core.observer.array-对Array的patch
date: 2019-04-26 18:31:18
categories: 
- 前端开发
tags: 
- 技术原理
---

## observer.array

Array的以下方法会改变原本的对象，所以需要对其进行patch处理，改造为响应式执行：

- push
- pop
- shift
- splice
- sort
- reverse

本质上是在执行原有API的基础上，添加两个步骤：

1. 如果有新元素进入，对新元素进行响应式改造
2. 触发一次更新通知

```typescript
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']

/**
 * 拦截Array中会改变Array对象自身的方法，在它们调用时，触发事件
 * 相当于对它做Monkey Patch
 */
methodsToPatch.forEach(function (method) {
  // 暂存原生的方法
  const original = arrayProto[method]
  // 改造原生方法（将其PropertyDescriptor的value覆盖）
  def(arrayMethods, method, function mutator(...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__

    // 处理有新元素插入的情况
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 如果有新的元素插入，将插入的新元素纳入观察
    if (inserted) ob.observeArray(inserted)

    // 通知发生了改变
    ob.dep.notify()
    return result
  })
})
```