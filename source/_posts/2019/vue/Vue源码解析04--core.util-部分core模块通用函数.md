---
title: Vue源码解析04--core.util-部分core模块通用函数
date: 2019-04-26 18:31:04
categories: 
- 前端开发
tags: 
- 技术原理
---

# Core/Util主体结构

- **next-tick**：`Vue.nextTick`的实现
- **props**：用于合法化prop，并加入observe
- **options**：Component构造函数中输入的options所需的相关处理
- **lang**：一些通用的检测与类型转换的封装
- **error**：针对`ViewModel`的错误信息的输出
- **perm**：Performance API的封装
- **env**：运行环境的检测
- **debug**：警告信息与错误栈的生成与输出，方便调试时排错



## 1. util/next-tick.js

`Vue.nextTick()`的实现，其优先级为：

1. `Promise`：micro
2. `setImmediate`：macro
3. `MessageChannel`：macro
4. `setTimeout`：macro

### 建模

对于`nextTick()`方法，它需要以下变量来建模：

- callbacks：回调队列，记录下一个tick要执行的所有回调
- pending：指示是否正在运行上一个tick的回调
- microTimerFunc：microtask的实现方式
- macroTimerFunc：macrotask的实现方式
- useMacroTask：指示是否使用macrotask

```typescript
// 回调队列，记录下一个tick的所有回调
const callbacks = []
// 用于指示是否正在运行上一个tick的callbacks
let pending = false

//依次执行所有的回调，复位pending，并清空callbacks队列
function flushCallBacks() {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for(let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let microTimerFunc
let macroTimerFunc
// 用于指示是否使用macroTimerFunc
let useMacroTask = false
```

### macroTimerFunc

决定使用MacroTask来执行的方式，优先级为：setImmediate > MessageChannel > setTimeout

```typescript
/**
 * 决定macroTimerFunc的实现。
 * 先用setImmediate（仅IE支持），没有就用MessageChannel，最后使用setTimeout
 */
// 先尝试setImmediate
if(typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
}
// 然后尝试MessageChannel
else if(typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) || 
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
}
// 最后尝试setTimeout
else {
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

### microTimerFunc

决定使用MicroTask执行的方式，首选Promise，不支持则转macroTimerFunc

```typescript
/**
 * 决定microTimerFunc的实现。
 * 先用Promise，没有就直接套用MacroTimerFunc
 */
// 首先尝试Promise
if(typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    
    // ios下要通过触发一个新的MacroTask，使得MicroTask在此之前执行
    if(isIOS) setTimeout(()=>{}， 0)
  }
} else {
  microTimerFunc = macroTimerFunc
}
```

### 核心：Vue.nextTick()实现

```typescript
/**
 * Vue.nextTick的实现
 */
function nextTick(cb?: Function, ctx?: Object) {
  let _resolve
  
  // 将回调以闭包形式加入堆栈，没有回调则以Promise的resolve作为回调
  callbacks.push(() => {
    if(cb) {
      try {
        cb.call(ctx)
      } catech(e) {
        handlerError(e, ctx, 'nextTick')
      }
    } else if(_resolve) {
      _resolve(ctx)
    }
  })
  
  // 非pending情况下，开始运行
  if(!pending) {
    pending = true
    // 检查使用哪种task来触发
    if(useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  
  // 回调为空，则尝试返回一个Promise来进行回调操作
  if(!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

### 其他的helper函数

```typescript
// 返回一个闭包，使得该fn强制在MacroTask中执行
function withMacroTask(fn: Function) {
  return fn._withTask || (fn._withTask = function() {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}
```



## 2. util/lang.js

提供一些通用方法

```typescript
// 检查某个字符串是否以$或_开头
function isReserved(str: String): boolean {
  const c = (str + '').charCodeAt(0)
  // 0x24为$，0x5F为_
  return c === 0x24 || c ===0x5F
}

// Object.defineProperty的封装，其中enumerable默认为false
function def(obj: Object, key: string, val: any, enumerable: boolean = false) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable,
    writable: true,
    configurable: true
  })
}

// 转换路径，即在一个Object中逐层向内寻找，在watch中有使用
const bailRE = /[^\w.$]/
export function parsePath(path: string): any {
  if(bailRE.test(path)) return
  
  const segments = path.split('.')
  return function(obj) {
    for(let i = 0; i < segments.length; i++) {
      if(!obj) return 
      obj = obj[segments[i]]
    }
    return obj
  }
}	
```



## 3. util/error.js

针对`ViewModel`的错误处理器，会输出错误栈以便进行追踪

```typescript
// 控制台输出错误
function logError(err, vm, info) {
  if(process.env.NODE_ENV !== 'production') {
    warn(`Error in ${info}: "${err.toString()}"`, vm)
  }
  
  if((inBrowser || inWeex) && typeof console !== 'undefined') {
    console.log(err)
  } else {
    throw err
  }
}

// 全局的ErrorHandler，从全局config中获取，并将其输出在控制台
function globalHandleError(err, vm, info) {
  if(config.errorHandler) {
    try {
      return config.errorHandler.call(null, err, vm, info)
    } catch(e) {
      logError(e, null, 'config.errorHandler')
    }
  }
  logError(err, vm, info)
}

// 处理错误，如果是vm中的错误，一层一层向上报出错误栈
function handleError(err: Error, vm: any, info: string) {
  if(vm) {
    let cur = vm
    // 组件从自身开始一层层向上报出错误
    while(cur = cur.$parent) {
      const hooks = cur.$options.errorCaptured
      if(hooks) {
        for(let i = 0; i < hooks.length; i++) {
          try {
            const capture = hooks[i].call(cur, err, vm, info) === false
            if(capture) return
          } catch(e) {
            globalHandleError(e, cur, 'errorCaptured hook')
          }
        }
      }
    }
  }
  globalHandleError(err, vm, info)
}
```



## 4. util/perm.js

`Performance` API相关的操作，用来记录操作发生的时间戳和计算某些操作所用的时间。

```typescript
export let mark
export let measure

if (process.env.NODE_ENV !== 'production') {
  const perf = inBrowser && window.performance
  // 检查浏览器是否支持Performance API
  if (
    perf &&
    perf.mark &&
    perf.measure &&
    perf.clearMarks &&
    perf.clearMeasures
  ) {
    // 记录发生的时间戳
    mark = tag => perf.mark(tag)
    // 计算两次记录的时间间隔，完成后消除相关记录和计算记录
    measure = (name, startTag, endTag) => {
      perf.measure(name, startTag, endTag)
      perf.clearMarks(startTag)
      perf.clearMarks(endTag)
      perf.clearMeasures(name)
    }
  }
}
```



## 5. util/env.js

`env`用于检测相关的系统环境。对于环境和设备类型的检测非常重要，可以根据其差异，来优化用户体验。

它包括了对以下环境变量和运行特性的检测：

- `__proto__`
- 运行环境：浏览器及其类型，Weex
- 被动的`eventListener`，即不能`preventDefault()`
- 是否为SSR
- `Vue Devtools`插件
- Symbol
- Set

```typescript
// 是否可以使用__proto__
export const hasProto = '__proto__' in {}

// 浏览器环境检测
export const inBrowser = typeof window !== 'undefined'
export const inWeex = typeof WXEnvironment !== 'undefined' && !!WXEnvironment.platform
export const weexPlatform = inWeex && WXEnvironment.platform.toLowerCase()
export const UA = inBrowser && window.navigator.userAgent.toLowerCase()
export const isIE = UA && /msie|trident/.test(UA)
export const isIE9 = UA && UA.indexOf('msie 9.0') > 0
export const isEdge = UA && UA.indexOf('edge/') > 0
export const isAndroid = (UA && UA.indexOf('android') > 0) || (weexPlatform === 'android')
export const isIOS = (UA && /iphone|ipad|ipod|ios/.test(UA)) || (weexPlatform === 'ios')
export const isChrome = UA && /chrome\/\d+/.test(UA) && !isEdge
// Firefox有独特的Object.prototype.watch()方法
export const nativeWatch = ({}).watch

// 检查是否支持被动的eventListener，即preventDefault()没有效果
export let supportsPassive = false
if (inBrowser) {
  try {
    const opts = {}
    Object.defineProperty(opts, 'passive', ({
      get () {
        supportsPassive = true
      }
    }: Object))
    window.addEventListener('test-passive', null, opts)
  } catch (e) {}
}

// 检查是否为Server Side Rendering，只能使用懒检查方式，因为SSR设置VUE_ENV晚于运行时
let _isServer
export const isServerRendering = () => {
  if (_isServer === undefined) {
    // 基本就是对Node.js环境的检测
    if (!inBrowser && !inWeex && typeof global !== 'undefined') {
      _isServer = global['process'].env.VUE_ENV === 'server'
    } else {
      _isServer = false
    }
  }
  return _isServer
}

// 检测开发工具（Chrome的Vue Devtools插件）
export const devtools = inBrowser && window.__VUE_DEVTOOLS_GLOBAL_HOOK__

// 检测某个class constructor是否原生支持
export function isNative (Ctor: any): boolean {
  return typeof Ctor === 'function' && /native code/.test(Ctor.toString())
}

// 是否支持ES6的Symbol
export const hasSymbol =  typeof Symbol !== 'undefined' && isNative(Symbol) &&
  typeof Reflect !== 'undefined' && isNative(Reflect.ownKeys)

// Set类型的检测，顺带简单的polyfill
let _Set
if (typeof Set !== 'undefined' && isNative(Set)) {
  _Set = Set
} else {
  _Set = class Set implements SimpleSet {
    set: Object;
    constructor () {
      this.set = Object.create(null)
    }
    has (key: string | number) {
      return this.set[key] === true
    }
    add (key: string | number) {
      this.set[key] = true
    }
    clear () {
      this.set = Object.create(null)
    }
  }
}
```



## 6. util/debug.js

 debug模式下的一些信息输出

```typescript
export let warn
export let tip
export let generateComponentTrace
export let formatComponentName

if(process.env.NODE_ENV !== 'production') {
	const hasConsole = typeof console !== 'undefined'
  
  // 去除-和_，转为Pascal命名法
  const classifyRE = /(?:^|[-_])(\w)/g
  const classify = str => str
    .replace(classifyRE, c => c.toUpperCase())
    .replace(/[-_]/g, '')
  
  // warn()方法，输出警告
  warn = (msg, vm) => {
    // vm存在时，输出组件错误栈
    const trace = vm ? generateComponentTrace(vm) : ''
    // 默认为console输出，除非特别指明了warnHandler
    if (config.warnHandler) {
      config.warnHandler.call(null, msg, vm, trace)
    } else if (hasConsole && (!config.silent)) {
      console.error(`[Vue warn]: ${msg}${trace}`)
    }
  }
  
  // tip()方法，输出可行的优化方式
  tip = (msg, vm) => {
    if (hasConsole && (!config.silent)) {
      console.warn(`[Vue tip]: ${msg}` + (vm ? generateComponentTrace(vm) : ''))
    }
  }
  
  // 格式化组件的名称
  // name有三个来源：options.name，options._componentTag，文件名（去除.vue后缀）
  formatComponentName = (vm, includeFile) => {
    if (vm.$root === vm) {
      return '<Root>'
    }
    // 多渠道获取options
    const options = typeof vm === 'function' && vm.cid != null
      ? vm.options
      : vm._isVue
        ? vm.$options || vm.constructor.options
        : vm || {}
    let name = options.name || options._componentTag
    const file = options.__file
    if (!name && file) {
      const match = file.match(/([^/\\]+)\.vue$/)
      name = match && match[1]
    }

    return (
      // 转为Pascal format
      (name ? `<${classify(name)}>` : `<Anonymous>`) +
      (file && includeFile !== false ? ` at ${file}` : '')
    )
  }
  
  // 将一个字符串重复n遍，通过按位判断的方式，时间复杂度降为O(logn)
  const repeat = (str, n) => {
    let res = ''
    while (n) {
      if (n % 2 === 1) res += str
      if (n > 1) str += str
      n >>= 1
    }
    return res
  }
  
  // 生成组件栈，方便输出时追踪
  generateComponentTrace = vm => {
    if (vm._isVue && vm.$parent) {
      const tree = []
      let currentRecursiveSequence = 0
      // 向父级递归，将合法的vm压入队列
      while (vm) {
        if (tree.length > 0) {
          const last = tree[tree.length - 1]
          if (last.constructor === vm.constructor) {
            currentRecursiveSequence++
            vm = vm.$parent
            continue
          } else if (currentRecursiveSequence > 0) {
            tree[tree.length - 1] = [last, currentRecursiveSequence]
            currentRecursiveSequence = 0
          }
        }
        tree.push(vm)
        vm = vm.$parent
      }
      return '\n\nfound in\n\n' + tree
        .map((vm, i) => `${i === 0 ? '---> ' : repeat(' ', 5 + i * 2)}
          ${Array.isArray(vm) 
             ? `${formatComponentName(vm[0])}... (${vm[1]} recursive calls)` 
             : formatComponentName(vm)
          }`,
        )
        .join('\n')
    } else {
      return `\n\n(found in ${formatComponentName(vm)})`
    }
  }
  
}
```

