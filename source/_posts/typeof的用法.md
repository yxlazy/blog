---
title: typeof的用法
date: 2021-06-30 16:11:59
tags: [javascript, lodash, 源码解析]
category: javascript
description: typeof 是javascript中的关键字，但如果你认为它只是用来判断类型，那你可就大错特错了
---
## 前言
`typeof` 的一般操作，就是用来判断某个变量是什么类型，从而进行下一步操作。但是最近发现`lodash`源码中`typeof`还有如此作用，特记录于此

## 常规操作
常规操作无非就是判断类型，如下
```js
typeof null === 'object' //true 注：特殊
typeof undefined === 'undefined' //true
typeof 111 === 'number' //true
typeof '字符串' === 'string' //true
typeof true === 'boolean' //true
typeof Symbol('foo') === 'symbol' //true
typeof 111n === 'bigint' //true
typeof {} === 'object' //true
typeof [] === 'object' //true
typeof (() => {}) === 'function' //true
```
## 非常规操作
我们知道，对于未声明的变量，只能执行一个操作，那就是使用`typeof`判断类型
```js
//let a = 1  //确保a没有声明
typeof a  //undefined
console.log(a) //报错: Uncaught ReferenceError: a is not defined
```
那怎么能让第二个语句不报错，有输出呢？往下看
```js
console.log(typeof a !== 'undefined' && a) //输出false
```
> `lodash`类似源码
>
> ```js
> /** Detect free variable `global` from Node.js. */
> const freeGlobal = typeof global === 'object' && global !== null && global.Object === Object && global
> 
> export default freeGlobal
> ```

