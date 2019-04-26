---
title: Vue源码解析03--core
date: 2019-04-26 18:31:03
categories: 
- 前端开发
tags: 
- 技术原理
---

## 1. 模块

- util：通用的方法。
- instance：Vue实例的定义与生成。
- observer：观察者，对组件实例属性的响应式改造，从而实现响应式更新。
- vdom：vnode的生成与更新机制。
- global-api：一些全局使用的API
- component：keep-alive



## 2. index.js

core的入口主要做这几件事：

- 初始化全局API
- 定义Vue.prototype的`$isServer`、`$ssrContext`属性
- 定义Vue的`FunctionalRenderContext`属性
- `Vue.version = '__VERSION__'`
- `export default Vue`



## 3. config.js

`core/config.js`用于定义Vue全局设置的类型和初始设置。

该初始设置绝大部分涉及到`shared`目录下的共享对象和函数。

在初始状态下绝大部分为空值或空函数，需要在运行时中，根据环境和具体设置来进行二次设定。