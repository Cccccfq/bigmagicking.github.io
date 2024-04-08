---
title: yield关键字
date: 2024-04-08 15:38:10
categories: 基础
tags: ['基础', '生成器']
---

## 简单介绍

- yield 是 ES6 的新关键字，使生成器函数执行暂停，yield 关键字后面的表达式的值返回给生成器的调用者。它可以被认为是一个基于生成器的版本的 return 关键字。
- yield 关键字实际返回一个 IteratorResult（迭代器）对象，它有两个属性，value 和 done，分别代表返回值和是否完成。
- yield 无法单独工作，需要配合 generator(生成器)的其他函数，如 next，懒汉式操作，展现强大的主动控制特性。

## 错误的调用

```javascript
/**
 * 错误的写法
 * 1. 在判断中调用next也会消耗next本身的迭代
 * 2. 下面的结果会消耗所有奇数项
 */
while (myArr.next().done !== true) {
  console.log(myArr.next())
}
```

## 一些说明

- yield 并不能直接生产值，而是产生一个等待输出的函数
- 除 IE 外，其他所有浏览器均可兼容（包括 win10 的 Edge）
- 某个函数包含了 yield，意味着这个函数已经是一个 Generator
- 如果 yield 在其他表达式中，需要用()单独括起来
- yield 表达式本身没有返回值，或者说总是返回 undefined(由 next 返回)
- next()可无限调用，但既定循环完成之后总是返回 undeinded

## 更深层次的理解 yield

```javascript
function* test(x) {
  let y = 2 * (yield x + 1)
  let z = yield y / 3
  return x + y + z
}
let a = test(5)
console.log(a.next()) // {value: 6, done: false}
console.log(a.next()) // {value: NaN, done: false}
console.log(a.next()) // {value: NaN, done: true}
let b = test(5)
console.log(b.next()) // {value: 6, done: false}
console.log(b.next(12)) // {value: 8, done: false}
console.log(b.next(13)) // {value: 42, done: true}
```

参考资料 1：https://zoyi14.smartapps.cn/pages/note/index?slug=36c74e4ca9eb&origin=share&_swebfr=1&_swebFromHost=baiduboxapp
