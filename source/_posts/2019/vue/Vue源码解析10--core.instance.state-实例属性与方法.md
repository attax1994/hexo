---
title: Vue源码解析10--core.instance.state-实例属性与方法
date: 2019-04-26 18:31:10
categories: 
- 前端开发
tags: 
- 技术原理
---

## 实例属性与方法

### 建模

一个Vue的实例在初始化属性与方法时，经历了以下步骤：

1. 初始化`props`，建立vm上的代理
2. 初始化`methods`，直接挂载到vm上，不设代理
3. 初始化`data`，建立vm上的代理。由于data一般情况下是函数，要做一次`getData()`执行来获得对象。
4. 初始化`computed`，直接挂载到vm上，不设代理，`vm._computedWatchers`记录它们的内部watcher
5. 初始化`watch`，不挂载，`vm._watchers`记录每个watch属性的watcher（也包括组件本身的watcher）

```typescript
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  
  // 初始化props
  if (opts.props) initProps(vm, opts.props)
  
  // 初始化methods
  if (opts.methods) initMethods(vm, opts.methods)
  
  // 初始化data，没有data时假定为空对象
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  
  // 初始化computed
  if (opts.computed) initComputed(vm, opts.computed)
  
  // 初始化watch
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```



### Proxy属性代理

为了可以方便地在vm上去访问这些属性和方法，也就是使用`vm.key`，而不是使用`vm._data.key`的方式，就需要在vm上提供一个该属性或方法的代理引用。

它们的指向依然是`_data`、`_props`或`_methods`等字段上的对应属性，只是为了使用上的方便而做了这个代理。

```typescript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop,
}

/**
 * 通过proxy函数将_data（或者_props等）上面的属性代理到vm上，
 * 从而直接通过 this.key 访问，而无需 this._data.key 的形式，
 * 本质上是改造vm上这个对应key的setter和getter，
 * 最终set和get操作的还是_data、_props等对象上面的属性
 */
export function proxy(target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter() {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter(val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```



### 初始化props

每个props的属性在初始化时，会经历以下步骤：

1. 将props的key缓存在`vm.$options._propKeys`，更新时遍历这个Array来取key，提高效率
2. 利用`core/util/props`提供的`validateProp`来验证prop
3. 检查有没有重名的字段
4. 通过`defineReactive()`方法，改造getter和setter，纳入观察
5. vm上对每个prop挂载代理

```typescript
function initProps(vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}

  // 将props的key缓存在vm.$options._propKeys，props更新时可以直接遍历这个Array，而不是枚举对象属性
  const keys = vm.$options._propKeys = []

  // 如果$parent为null或undefined，说明是根节点
  const isRoot = !vm.$parent
  // 非root节点，可以在最后将shouldConvert赋值为true
  if (!isRoot) {
    toggleObserving(false)
  }

  /**
   * 将props上的属性代理到vm上
   */
  for (const key in propsOptions) {
    // 将key存入vm.$options._propKeys
    keys.push(key)
    // 验证prop
    const value = validateProp(key, propsOptions, propsData, vm)

    /**
     * 改造getter和setter，纳入观察
     */
    if (process.env.NODE_ENV !== 'production') {
      // 检查是否为保留的字段
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
        config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm,
        )
      }
      // 在setter中提示不要在子组件中直接修改props的属性，而是让父组件传入
      defineReactive(props, key, value, () => {
        if (vm.$parent && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm,
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }

    // Vue.extend()期间，静态props已经被代理到组件原型上，因此只需要代理props
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```



### 初始化methods

methods 中定义的方法不需要做响应式处理，因此也不需要做代理，直接验证没有重名的情况下，挂载到 vm 上，并 bind 它的 this 指针为 vm。

```typescript
function initMethods(vm: Component, methods: Object) {
  const props = vm.$options.props
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      // 保证不是null或者undefined
      if (methods[key] == null) {
        warn(
          `Method "${key}" has an undefined value in the component definition. ` +
          `Did you reference the function correctly?`,
          vm,
        )
      }
      // 保证不和props的属性重名
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm,
        )
      }
      // 保证不是vm保留字段
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`,
        )
      }
    }
    // 方法可以直接挂载，无需proxy，注意bind它的this指针到vm实例
    vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
  }
}
```



### 初始化data

由于data通常是一个function，所以要使用`getData()`方法来得到它的返回值，作为初始值。它的过程为：

1. 在定义了data的情况下，调用`getData()`获得返回值，作为data初始值
2. 检查是否重名
3. vm上挂载data各个属性的代理
4. 使用`observe`方法，将data加入观察

```typescript
function initData(vm: Component) {
  let data = vm.$options.data
  /**
   * data通常应当为function，执行后得到返回对象
   */
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm,
    )
  }

  /**
   * 将data代理到vm实例上，
   */
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    // 保证不与methods中的属性重名
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm,
        )
      }
    }
    // 保证不与props中的属性重名
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm,
      )
    }
    // 保证不是vm的保留字段
    else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }

  // 将data加入观察
  observe(data, true /* asRootData */)
}

export function getData(data: Function, vm: Component): any {
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}
```



### 初始化computed

computed的取值本质上是基于脏值检查的，只是get的触发时机通常由data和props控制。

computed的创建过程为：

1. 提取每个属性的`get`和`set`方法，仅传入function则认为只有`get`
2. 建立其内部的watcher，保存在`vm._computedWatchers`中
3. 检查是否重名
4. 改造computed属性的getter，使用`watcher.depend()`来收集依赖，`watcher.evaluate()`来做脏值检查。
5. 直接挂载到vm实例上，无需创建额外的代理

```typescript
const computedWatcherOptions = {computed: true}

function initComputed(vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  const isSSR = isServerRendering()

  /**
   * 计算属性可能是一个function，也有可能设置了get以及set的对象，
   */
  for (const key in computed) {
    const userDef = computed[key]
    /**
     * 只是一个function，说明只有getter，否则就要从对象中提取get
     */
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm,
      )
    }

    if (!isSSR) {
      /**
       * 为computed的属性创建内部watcher，保存在vm实例的_computedWatchers中
       * 这里的computedWatcherOptions参数传递了computed: true
       */
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions,
      )
    }

    // 避免重复定义同名的属性
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}

/**
 * 定义单个computed属性
 * @param target
 * @param key
 * @param userDef
 */
export function defineComputed(target: any, key: string, userDef: Object | Function) {
  // SSR情况下无需watch
  const shouldCache = !isServerRendering()

  /**
   * 单传function，就是只有get
   * getter会被进行改造，
   * 调用对应watcher的depend()方法进行依赖收集，调用evaluate()方法返回最新的值
   */
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : userDef
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : userDef.get
      : noop
    sharedPropertyDefinition.set = userDef.set
      ? userDef.set
      : noop
  }
  // 没有定义setter的时候，给一个默认的setter，提示不能对它进行set
  if (process.env.NODE_ENV !== 'production' &&
    sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this,
      )
    }
  }
  // 直接将其定义到实例上
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

/**
 * 创建计算属性的getter
 * @param key
 * @return value
 */
function createComputedGetter(key) {
  return function computedGetter() {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      // 依赖收集
      watcher.depend()
      // 脏检查，获取最新的数值
      return watcher.evaluate()
    }
  }
}
```



### 初始化watch

`watch`字段只是一个语法糖，本质上是利用`Vue.prototype.$watch()`方法来创建的。

```typescript
Vue.prototype.$watch = function (expOrFn: string | Function, cb: any, options?: Object): Function {
    const vm: Component = this
    // 对象形式的callback，让createWatcher()去解包
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }

    options = options || {}
    options.user = true

    // 给它建立一个watcher，做监测，会在vm._watchers中加入这个watcher
    const watcher = new Watcher(vm, expOrFn, cb, options)

    // 设置immediate: true的时候会立即执行一次
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }

    // 返回unwatch()的方法，用于解除这个watch，停止触发回调
    return function unwatchFn() {
      watcher.teardown()
    }
  }
```

watch的handler支持`Function | Object`和`Array<Function | Object>`形式。

```typescript
function initWatch(vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}

/**
 * 创建单个watch，利用Vue.prototype.$watch()方法来进行监测
 * @param vm
 * @param expOrFn
 * @param handler
 * @param options
 * @return {Function}
 */
function createWatcher(vm: Component, expOrFn: string | Function, handler: any, options?: Object) {
  // 对对象形式的handler做解包
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  // 对字符串形式的handler，认为它是一个key，从vm上找到对应的method
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```



### State的混入

向Vue提供原型属性和方法，包括：

- `$data`和`$props`属性
- `$set()`和`$delete()`方法
- `$watch()`方法

```typescript
export function stateMixin(Vue: Class<Component>) {
  const dataDef = {}
  dataDef.get = function () {
    return this._data
  }
  const propsDef = {}
  propsDef.get = function () {
    return this._props
  }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function (newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this,
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  // 提供$data和$props属性，作为_data和_props的代理
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  // 向响应式对象添加响应式的属性
  Vue.prototype.$set = set
  // 与set对立，删除对象属性
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function() {/* ... */}
}
```

