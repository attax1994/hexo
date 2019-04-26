---
title: Vue源码解析07--core.instance-Vue实例
date: 2019-04-26 18:31:07
categories: 
- 前端开发
tags: 
- 技术原理
---

## 1. index.js

`src/instance/index.js`主要做如下几件事

1. `initMixin()`混入（mixin）初始化设置，包括：
   1. ` initLifeCycle(vm)`
   2. `initEvents(vm)`
   3. `initRender(vm)`
   4. `callHook(vm, 'beforeCreate')`
   5. `initInjections(vm)`
   6. `initState(vm)`
   7. `initProvide(vm)`
   8. `callHook(vm, 'created')`   
2. `stateMixin()`混入状态
3. `eventsMixin()`混入事件机制
4. `lifecycleMixin()`混入生命周期
5. `renderMixin()`混入渲染器



 ## 2. init.js

### `initMixin(Vue)`

`initMixin(Vue)`混入初始化设置，改造Vue.prototype，赋予`_init(options)`方法。

需要`initInternalComponent(vm, options)`和`resolveConstructorOptions(Ctor)`的辅助。

从该段源码分析，可以得知一个`Component`在`create`阶段的执行步骤：

1. 设置Component的uid，作为唯一标识
2. 非prod环境记录开始点位
3. `vm._isVue = true`，防止被纳入observe
4. 如果是组件，初始化其内部设置；不是组件则执行设置的混入（递归混入上层组件的设置）
5. 非prod环境尝试Proxy API
6. `vm._self = vm`暴露自身
7. 初始化操作
  1. 初始化生命周期、事件机制、Render
  2. **触发beforeCreate钩子，进入生命周期**
  3. 初始化Injection、状态、Provide
  4. **触发created钩子，创建完成**
8. 非prod环境记录结束点位，计算创建时间
9. 对组件进行挂载

```typescript
// 每次自增1，保持每个component有唯一的_uid
let uid = 0

function initMixin(Vue: Class<Component>) {
  // 原型方法，初始化
  Vue.prototype._init = function(options: Object) {
    const vm: Compnent = this
    // 设置component的id
    vm._uid = uid++
    
    let startTag, endTag
    // 检查环境，debug模式下使用
    if(process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      // 记录初始化开始
      mark(startTag)
    }
    
    // 防止它被observe
    vm._isVue = true
    
    // 如果是组件，就初始化内部设置
    if(options && options._isComponent) {
      initInternalComponent(vm, options)
    }
    // 不是组件，混入设置
    else {
      vm.$options = mergeOptions(
      	resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    
    // 非生产环境下，尝试Proxy API
    if(process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    
    // 暴露自身
    vm._self = vm
    // 初始化生命周期、事件机制、Render
    initLifeCycle(vm)
    initEvents(vm)
    initRender(vm)
    // 触发beforeCreate钩子，进入生命周期!!!
    callHook(vm, 'beforeCreate')
    // 初始化Injection、状态、Provide
    initInjections(vm)
    initState(vm)
    initProvide(vm)
    // 触发created钩子，创建完成
    callHook(vm, 'created')

    if(process.env.NODE_ENV !== 'production' && config.performance && mark) {
      // Component的名称
      vm._name = formatComponentName(vm, false)
      // 记录初始化结束
      mark(endTag)
      // 计算初始化所需的时间，便于性能分析
      measure(`vue ${vm._name} init`, startTag, endTag)
    }
    
    // 如果有DOM节点，就将其挂载上去
    if(vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```



### `initInternalComponent(vm, options)`

`initInternalComponent(vm, options)`用于初始化组件内部的设置。

```typescript
function initInternalComponent(vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode
  
  // 拷贝父组件的设置
  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```



### `resolveConstructorOptions(Ctor)`

`resolveConstructorOptions(Ctor)`用于处理构造器的设置（主要是递归解决继承的设置）。

需要`resolveModifiedOptions(Ctor)`和`dedupe(latest, extended, sealed)`的辅助处理

```typescript
function resolveConstructorOptions(Ctor: Class<Component>) {
  let options = Ctor.options
  if(Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    
    // 上级的设置发生改变，需要刷新
    if(superOptions !== cachedSuperOptions) {
      Ctor.superOptions = superOptions
      
      // 获取修改过的设置组成的键值对
      const modifiedOptions = resolveModifiedOptions(Ctor)
      if(modifiedOptions) {
        // 混入修改过的设置
        extend(Ctor.extendOptions, modifiedOptions)
      }
      
      // 合并设置
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if(options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}

// 获取改变的设置，返回其构成的对象
function resolveModifiedOptions(Ctor: Class<Component>) {
  let modified
  const {options: latest, extendOptions: extended, sealedOptins: sealed} = Ctor
  for(const key in latest) {
    if(latest[key] !== sealed[key]) {
      if(!modified) modified = {}
      // 考虑value为Array的情况
      modified[key] = dedupe(latest[key], extended[key], sealed[key])
    }
  }
  return modified
}

// 对Array的情况也进行处理，防止忽略
function dedupe(latest, extended, sealed) {
  // 如果是Array，则需要遍历处理
  if(Array.isArray(latest)) {
    const res = []
    sealed = Array.isArray(sealed) ? sealed : [sealed]
    extended = Array.isArray(extended)? extended : [extended]
    
    // 将非sealed设置，但是extended的设置加入到结果中
    for(let i = 0; i < latest.length; i++) {
      if(extended.indexOf(latest[i]) >= 0 || sealed.indexOf(latest[i]) < 0) {
        res.push[latest[i]]
      }
    }
    return res
  }
  // 不是Array，直接返回latest
  else {
    return latest
  }
}
```

