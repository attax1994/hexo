---
title: Vue源码解析14--core.observer.index-Observer类
date: 2019-04-26 18:31:14
categories: 
- 前端开发
tags: 
- 技术原理
---

## Observer类

Observer类，附着到每个被观察的对象。

 一旦被添加，observer会将目标对象的属性转换为getter/setters形式，其`dep`用于收集依赖和发出更新通知。

它包括了对以下数据的处理：

- 基本类型，数组，对每一个成员加入观察。同时在数组的原型上，对数组的部分原生方法进行修改（push，pop等），从而让它们在调用时发出更新通知。
- 对象，直接对键进行遍历并绑定。

```typescript
/**
 * 用于指示是否要进行观察
 * 有时候在component的update运算中，要停止观察
 */
export let shouldObserve: boolean = true

export function toggleObserving(value: boolean) {
  shouldObserve = value
}

export class Observer {
  value: any
  dep: Dep
  // 将这个Object作为根$data的vm数量
  vmCount: number

  constructor(value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    // 将这个实例作为目标对象的__ob__属性
    def(value, '__ob__', this)

    if (Array.isArray(value)) {
      /**
       * 如果是数组，对数组的部分原生方法进行修改，从而当它们被调用时，会发出更新的通知。
       * 如果当前浏览器支持__proto__属性，则直接覆盖当前数组对象原型上的原生数组方法，如果不支持该属性，则直接覆盖数组对象的原型。
       */
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      /**
       * 如果是对象，直接进行遍历绑定
       */
      this.walk(value)
    }
  }

  /**
   * 遍历每个属性，将他们转换成getter/setter形式。
   * 这个方法只应在value的类型为Object的时候调用
   */
  walk(obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * 将一个Array的每个item加入观察
   */
  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

/**
 * 如果支持__proto__属性，直接将target的__proto__替换为src
 */
function protoAugment(target, src: Object, keys: any) {
  target.__proto__ = src
}

/**
 * 如果不支持__proto__属性，遍历src的每个键值对，混入到target上去
 */
function copyAugment(target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```



### `Observe`函数

尝试给一个对象创建observer实例，挂载在`vm.__ob__`。如果成功创建，返回新的observer，如果该对象已经有observer，返回原observer。

创建的条件：

1. 必须是普通对象或Array，可以添加新属性（没有被`Object.preventExtension()`处理过）
2. `shouldObserve`为true
3. 不是服务端渲染（SSR）
4. 不是Vue实例（Vue实例要结合`instance`的创建过程去达到响应式）

```typescript
export function observe(value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```



### `defineReactive`函数

用于为对象定义一个响应式属性。这个被定义的属性会获得一个内置的`Dep`，用于收集它的依赖，并发布它的更新通知。

`Observer`中value的每个属性，以及`Vue.set()`添加的新属性都会经历它的改造

```typescript
export function defineReactive(obj: Object, key: string, val: any, customSetter?: ?Function, shallow?: boolean) {
  // 给这个属性提供一个内置的Dep
  const dep = new Dep()

  // 检查PropertyDescriptor的configurable
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // 取出它原有的getter和setter，在此基础上打patch
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 深度模式下，给这个属性也添加observer
  let childOb = !shallow && observe(val)

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    /**
     * 先执行原本的getter，然后收集依赖
     */
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    /**
     * 先执行原本的getter，检查新的值是否发生了改变。
     * 然后执行原本的setter，最后发布通知
     */
    set: function reactiveSetter(newVal) {
      const value = getter ? getter.call(obj) : val
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    },
  })
}
```



### Vue.prototype.set()

Vue.set和vm.$set的实现，在target上设置一个响应式的属性。

如果是新增的属性，使用`defineReactive()`来执行添加，并立即触发一次通知。

```typescript
export function set(target: Array<any> | Object, key: any, val: any): any {
  if (process.env.NODE_ENV !== 'production' && (isUndef(target) || isPrimitive(target))) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }

  /**
   *  Array用splice()方法插入新的值，注意Array.prototype.splice已经被改造过了，会触发一次更新通知
   */
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  /**
   * 对于已经存在的实例属性，直接赋值，会触发setter
   */
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.',
    )
    return val
  }
  // 没有observer的，直接返回
  if (!ob) {
    target[key] = val
    return val
  }
  /**
   * 新增的属性，使用defineReactive()来实现响应式，并立即通知一次更新
   */
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}
```



### Vue.prototype.delete

Vue.delete和vm.$delete的实现。删除某个属性，并且触发更新通知。

```typescript
/**
 * 
 * 
 */
export function del(target: Array<any> | Object, key: any) {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot delete reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }

  /**
   * Array用splice()删除这个成员，注意Array.prototype.splice已经被改造过了，会触发一次更新通知
   */
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.splice(key, 1)
    return
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid deleting properties on a Vue instance or its root $data ' +
      '- just set it to null.',
    )
    return
  }
  if (!hasOwn(target, key)) {
    return
  }
  /**
   * 删除这个属性，如果target存在observer，那么需要触发一次更新通知
   */
  delete target[key]
  if (!ob) {
    return
  }
  ob.dep.notify()
}
```

