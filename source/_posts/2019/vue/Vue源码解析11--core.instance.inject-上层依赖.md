---
title: Vue源码解析11--core.instance.inject-上层依赖
date: 2019-04-26 18:31:11
categories: 
- 前端开发
tags: 
- 技术原理
---

## 依赖注入

`provide`和`inject`需要配套使用。

provide用于高阶组件向所有内部组件注入依赖，依赖对象保存在高阶组件的`vm._provided`字段上。

inject用于子组件从最近的一级上获得对应依赖。

### `initProvide`

初始化provide，记录在vm._provided上。provide可以传入function，需要调用一下获得结果。

依赖的名称一般为string，也可以使用Symbol，但是需要浏览器环境的支持。

```typescript
export function initProvide(vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```



### `initInjections`

初始化injections。

对于每个依赖，从组件自身开始，遵循就近原则，逐层向上，在上级组件的`_provided`属性中寻找对应依赖。

```typescript
export function initInjections(vm: Component) {
  // 寻找依赖
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    // 将每个依赖改造为响应式的，并提示不要进行直接的修改
    Object.keys(result).forEach(key => {
      if (process.env.NODE_ENV !== 'production') {
        defineReactive(vm, key, result[key], () => {
          warn(
            `Avoid mutating an injected value directly since the changes will be ` +
            `overwritten whenever the provided component re-renders. ` +
            `injection being mutated: "${key}"`,
            vm,
          )
        })
      } else {
        defineReactive(vm, key, result[key])
      }
    })
    toggleObserving(true)
  }
}

/**
 * 寻找对应的依赖
 * @param inject
 * @param vm
 * @return {any}
 */
export function resolveInject(inject: any, vm: Component): ?Object {
  if (inject) {
    const result = Object.create(null)
    // 过滤出可枚举的属性
    const keys = hasSymbol
      ? Reflect.ownKeys(inject).filter(key => Object.getOwnPropertyDescriptor(inject, key).enumerable)
      : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = inject[key].from
      let source = vm
      /**
       * 从自身开始，遵循就近原则，逐层向上，在上级组件的_provided属性中寻找对应依赖
       */
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }

      /**
       * 如果没找到，检查该依赖有没有设置default，如有则使用default
       * 如果没有default，就要提示没有找到依赖
       */
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        } else if (process.env.NODE_ENV !== 'production') {
          warn(`Injection "${key}" not found`, vm)
        }
      }
    }
    return result
  }
}
```