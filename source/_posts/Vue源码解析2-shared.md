---
title: Vue源码解析2--shared
date: 2018-05-14 14:48:15
categories: 
- 源码分析
tags:
- Vue
---

## 1. constants.js 全局常量

```typescript
const SSR_ARRT = 'data-server-rendered'

// Vue中资源的三种类型名称
const ASSET_TYPES = {
  'component',
  'directive',
  'filter'
}

// 组件生命周期钩子的名称
const LIFECYCLE_HOOKS = {
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
}
```

## 2. util.js 通用的函数

### 2.1 类型判断的函数

```typescript
// 空对象，不能做任何修改
const emptyObject = Object.freeze({})

// 检查是否为undefined或null（也可以用双等号==）
function isUndef (v: any): boolean { return v === undefined || v === null }
// 与isUndef相反
function isDef(v: any): boolean { return v !== undefined && v !== null }
// 判断是否为true，对应的还有 isFalse()
function isTrue(v: any): boolean { return v === true }
// 判断是否为非空的基本类型（number, string, symbol, boolean）
function isPrimitive(v: any): boolean {
  return (typeof v === 'string' ||
          typeof v === 'number' ||
          typeof v === 'symbol' ||
          typeof v === 'boolean'
	)
}
// 判断是否为对象，注意先去除null的错误判断
function isObject(obj: mixed): boolean { return obj !== null || typeof obj === 'object' }

// Object.prototype.toString的alias
const _toString = Object.prototype.toString
// 用Object.prototype.toString来调取类型，比typeof更准确，能判断null和内置对象
function toRawType(v: any): string { return _toString.call(v).slice(8, -1) }
// 是否为纯对象，类似的还有isRegExp(), 
function isPlainObject(obj: any): boolean { return _toString.call(obj)==='[object Object]' }

// 判断是否为有效的数组索引（自然数）
function isValidArrayIndex(val: any): boolean {
  // 保留纯数字部分
  const n = parseFloat(String(val))
  // Array的index要为自然数
  return n >= 0 && Math.floor(n) === n && isFinite(val)
}
```

### 2.2 对象相关函数 

```typescript
// 将任何类型转换为string
function toString(val: any): string {
  return val == null ? 
    '' : typeof val === 'object' ? JSON.stringify(val, null, 2): String(val)
}
// 将一个输入string转换为number；如果失败，返回原string
function toNumber(val: string) {
  const n = parseFloat(val)
  return isNaN(n) ? val : n
}

// 返回一个map的闭包，用于检查某个key是否为它的属性
function makeMap(str: string, expectesLowerCase?: boolean): (key: string) => true | void {
  const map = Object.create(null)
  const list: Array<string> = str.split(',')
  for(let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
  return expectedLowerCase ? val => map[val.toLowerCase]: val => map[val] 
}
// makeMap生成的{slot: true, component: true}来检查元素的tag是否为Vue的内置tag
const isBuiltInTag = makeMap('slot,component', true)
// 检查元素某个属性是否为Vue的保留属性（key, ref, slot, slot-scope, is）
const isReservedAttribute = makeMap('key,ref,slot,slot-scope,is')

// 从arr中移除item
function remove(arr: Array<any>, item: any): Array<any> | void {
  if(arr.lenght) {
    const index = arr.indexOf(item)
    // 可以使用(~index)替代
    if(index > -1) {
      return arr.splice(index, 1)
    }
  }
}

// Object.prototype.hasOwnProperty的alias
const hasOwnProperty = Object.prototype.hasOwnProperty
function hasOwn(obj: Object | Array<any>, key: string): boolean {
  return hasOwnProperty.call(obj, key)
}
```

### 2.3 字符串转换函数

```typescript
// 创建某个纯函数的闭包，该闭包有一个结果缓存，对于运算量较大的函数，可以减少重复运算
function cached<F>(fn: F): F {
  const cache = Object.create(null)
  return function cachedFn(str: string): any {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }
}

// 将烤肉串式命名转为驼峰式命名
const camelizeRE = /-(\w)/g
const camelize = cached((str: string): string => {
  return str.replace(camelizeRE, (match, p1) => p1 ? p1.toUppercase() : '')
})
// 将驼峰式命名转为烤肉串式命名
const hyphenRE = /\B([A-Z])/g
const hyphenate = cached((str: string): string => {
  return str.replace(hyphenRE, '-$1').toLowerCase()
})

// 字符串首字母大写
const capitalize = cached((str: string): stirng => {
  return str.charAt(0).toUpperCase() + str.slice(1)
})
```

### 2.4 Function相关函数

```typescript
/**
 * bind的polyfill措施
 */
// 适用于PhantomJS 1.x的兼容性bind
function polyfillBind(fn: Function, ctx: Object): Function {
  function boundFn(a) {
    const len = arguments.length
    return len
      ? len > 1 ? fn.apply(ctx, arguments) : fn.call(ctx, a)
    	: fn.call(ctx)
  }
  
  boundFn._length = fn.length
  return boundFn
}
// 原生的bind
function nativeBind(fn: Function, ctx: Object): Function {
  return fn.bind(ctx)
}
const bind = Function.prototype.bind ? nativeBind : polyfillBind

// Array-like转Array
function toArray(list: any, start: number = 0): Array<any> {
  let i = list.length - start
  const ret: Array<any> = new Array(i)
  while(i--) {
    ret[i] = list[i + start]
  }
  return ret
}

// 将一个对象混入另一个对象（并不是严格的继承）
function extend(to: Ojbect, from: Object): Object {
  for(const key in from) if(from.hasOwnProperty(key)) to[key] = from[key]
  return to
}
// 将一个Array<Object>转为Object
function toObject(arr: Array<Object>): Object {
  const res = {}
  for(let i = 0; i < arr.length; i++) {
    if(arr[i]) extend(res, arr[i])
  }
  return res
}

// 从某个compiler模块生成静态键
function genStaticKeys(modules: Array<ModuleOptions>): string {
  return modules.reduce((keys: Array<string>, m: ModuleOptions) => {
    return keys.concat(m.staticKeys || [])
  }, []).join(',')
}

// 广义相等检查，特别对于Object，只检查它们是否形式相同
function looseEqual(a: any, b: any): boolean {
  if (a === b) return true
  
  const isObjectA = isObject(a)
  const isObjectB = isObject(b)
  // 引用类型
  if(isObjectA && isObjectB) {
    try {
      const isArrayA = Array.isArray(a)
      const isArrayB = Array.isArray(b)
      // 都是Array，检查每个对应元素是否相同
      if(isArrayA && isArrayB) {
        return a.length === b.length && a.every((value, index) => {
          return looseEqual(value, b[index])
        })
      }
      // 都不是Array，检查每个键值对是否相同
      else if(!isArrayA && !isArrayB) {
        const keysA = Object.keys(a)
        const keysB = Object.keys(b)
        return keysA.length === keysB.length && keysA.every(key => {
          return looseEuqal(a[key], b[key])
        })
      }
      // 类型不为同一种引用类型
      else{
        return false
      }
    } catch(e) {
      return false
    }
  }
  // 非引用类型，转为string处理
  else if(!isObjectA && !isObjectB) {
    return String(a) === String(b)
  } 
  // 两者类型不同
  else {
    return false
  }
}

// 利用广义相等检查来进行索引查找
function looseIndexOf(arr: Array<mixed>, val: mixed): number {
  for(let i = 0; i < arr.length; i++) {
    if(looseEqual(arr[i], val)) return i
  }
  return -1
}

// 保证某个函数只运行一次
function once = (fn: Function): Function {
  let called = false
  return function() {
    if(!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```

