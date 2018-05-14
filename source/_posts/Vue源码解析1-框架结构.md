---
title: Vue源码解析1--框架结构
date: 2018-05-14 14:39:23
categories: 
- 源码分析
tags:
- Vue
---

Vue/src：
- complier：编译器，用于对 Vue 的 HTML 模板进行编译，生成浏览器可以识别的 HTML文件。
- core：核心部分
- platforms：平台适配，包括 web 平台和 weex 平台
- server：SSR（Server Side Rendering）服务端渲染
- sfc：单文件组件（Single File Component）
- shared：通用的 constants 和 util

Vue/flow：各模块的类型声明（针对flow）

Vue/types：各模块的类型声明（针对TypeScript）


