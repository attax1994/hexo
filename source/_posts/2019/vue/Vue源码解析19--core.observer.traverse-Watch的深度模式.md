---
title: Vue源码解析19--core.observer.traverse-Watch的深度模式
date: 2019-04-26 18:31:19
categories: 
- 前端开发
tags: 
- 技术原理
---

## Traverse

在Watcher设置为deep: true的模式下使用。

递归每一个对象或数组，触发他们转换过的getter，这样每个成员都会被依赖收集，形成深度的依赖关系。

使用一个Set是为了记录已经处理过的`depId`，避免重复触发某个`dep`

```typescript
const seenObjects = new Set()

export function traverse (val: any) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}

function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }

  // 记录处理depId，跳过已经处理过的
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }

  /**
   * 遍历每个成员，递归处理，从而对成员为引用类型的情况做深度处理。
   * 注意在传参时使用了键访问，此时便会触发getter。
   */
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}

```

