---
title: 生成器
date: 2024-04-07 22:22:22
categories: 基础
tags: ['基础', '生成器']
---

## Generators 简单介绍

一个简单的 Generator 函数示例

```javascript
function* example() {
  yield 1
  yield 2
  yield 3
}
var iter = example()
iter.next() //{value:1，done:false}
iter.next() //{value:2，done:false}
iter.next() //{value:3，done:false}
iter.next() //{value:undefined，done:true}
```

上述代码中定义了一个生成器函数，当调用生成器函数 example() 时，并非立即执行该函数，而是返回一个生成器对象。每当调用生成器对象的.next() 方法时，函数将运行到下一个 yield 表达式，返回表达式结果并暂停自身。当抵达生成器函数的末尾时，返回结果中 done 的值为 true，value 的值为 undefined。我们将上述 example() 函数称之为生成器函数，与普通函数相比二者有如下区别:

- 普通函数使用 function 声明，生成器函数用 function\*声明
- 普通函数使用 return 返回值，生成器函数使用 yield 返回值
- 普通函数是 run to completion 模式，即普通函数开始执行后，会一直执行到该函数所有语句完成，在此期间别的代码语句是不会被执行的；生成器函数是 run-pause-run 模式，即生成器函数可以在函数运行中被暂停一次或多次，并且在后面再恢复执行，在暂停期间允许其他代码语句被执行

## generator 源码简析

```javascript
// 示例
function* helloWorldGenerator() {
  yield 'hello'
  yield 'world'
  return 'ending'
}

var hw = helloWorldGenerator()

console.log(hw.next()) // {value: "hello", done: false}
console.log(hw.next()) // {value: "world", done: false}
console.log(hw.next()) // {value: "ending", done: true}
console.log(hw.next()) // {value: undefined, done: true}
// 编译结果

var _marked = /*#__PURE__*/ regeneratorRuntime.mark(helloWorldGenerator)

function helloWorldGenerator() {
  return regeneratorRuntime.wrap(function helloWorldGenerator$(_context) {
    while (1) {
      switch ((_context.prev = _context.next)) {
        case 0:
          _context.next = 2
          return 'hello'

        case 2:
          _context.next = 4
          return 'world'

        case 4:
          return _context.abrupt('return', 'ending')

        case 5:
        case 'end':
          return _context.stop()
      }
    }
  }, _marked)
}
```

以上就是 generator 的编译结果，乍一看代码并不多，看内部的逻辑也是用 switch case 实现的，大体和我们的思路相同，但是细看会发现，有几个东西不认识，regeneratorRuntime 是个什么鬼，mark 和 wrap 又是个啥？想要弄懂原理，我们有必要搞清楚这些东西都是什么。

先说一下 regenerator，这个是 facebook 旗下的一个工具，用来编译 es6 的 generator，如果想看到完整的 generator 代码，需要去这个工具里去看源码。

### mark 函数

我们先查看完整的 mark 函数源码

```javascript
runtime.mark = function (genFun) {
  genFun.__proto__ = GeneratorFunctionPrototype
  genFun.prototype = Object.create(Gp)
  return genFun
}
```

这部分代码比较少，虽然又牵扯到了我们两个不知道的东西，GeneratorFunctionPrototype 和 Gp，但是其实也无关紧要，从这部分代码中我们可以看出，mark 函数其实就是对我们传入的 genFun 绑定了一系列的原型，继承了一些属性方法（想查看具体继承了什么可以查阅上面提到的 regenerator）。

### wrap 函数

```javascript
function wrap(innerFn, outerFn, self) {
  var generator = Object.create(outerFn.prototype)
  var context = new Context([])
  generator._invoke = makeInvokeMethod(innerFn, self, context)

  return generator
}
```

从这段代码中可以看出，wrap 做的东西比较简单，创建了一个 generator，new 了一个 context 对象，再给 generator 绑定了一个 invoke 方法，该方法是 makeInvokeMethod，接收了三个参数，innerFn，self 以及 context，最后再把 generator 返回。
到这里我们先联系一下最开始我们用 babel 编译的结果，helloWorldGenerator 被分成了两部分，一部分是外层的 helloWorldGenerator 函数，另一部分是用 wrap 包裹的 helloWorldGenerator$函数，而 wrap 函数接受的是内层函数，所以在 wrap 定义中，第一个参数是 innerFn，也就是内层函数的意思。

但是这部分代码里，我们还有一些东西不知道，Context 类和 makeInvokeMethod 函数还需要继续阅读源码，我们先看 context。

```javascript
var ContinueSentinel = {}

var context = {
  done: false,
  method: 'next',
  next: 0,
  prev: 0,
  abrupt: function (type, arg) {
    var record = {}
    record.type = type
    record.arg = arg

    return this.complete(record)
  },
  complete: function (record, afterLoc) {
    if (record.type === 'return') {
      this.rval = this.arg = record.arg
      this.method = 'return'
      this.next = 'end'
    }

    return ContinueSentinel
  },
  stop: function () {
    this.done = true
    return this.rval
  },
}
```

**makeInvokeMethod 方法的源码**

```javascript
var ContinueSentinel = {}

function makeInvokeMethod(innerFn, self, context) {
  // 状态设置为start
  var state = 'start'
  return function invoke(method, arg) {
    // 已完成
    if (state === 'completed') {
      return { value: undefined, done: true }
    }
    context.method = method
    context.arg = arg
    // 执行中
    while (true) {
      state = 'executing'
      var record = {
        type: 'normal',
        arg: innerFn.call(self, context), // 执行下一步，并获取状态（其实就是switch里return的值）
      }
      if (record.type === 'normal') {
        // 判断是否已经执行完成
        state = context.done ? 'completed' : 'yield'
        // 	ContinueSentinel其实是一个空对象，record.arg === {}则跳过return进入下一个循环，那什么什么record.arg会为空对象呢，答案是没有后续yield语句或已经return 的时候，也就是switch反悔了空值的情况
        if (record.arg === ContinueSentinel) {
          continue
        }
        return {
          value: record.arg,
          done: context.done,
        }
      }
    }
  }
}
```

我们把这两部分代码连着解读，当函数一开始执行的时候，我们把状态设置为 start，该状态被内部返回的 invoke 方法占用，所以不会被销毁，invoke 内部先判定 state 是否是 completed 状态，如果是直接返回最终状态，如果不是我们把 invoke 方法传入 method 和 arg 赋值给上下文对象 context。状态不为结束时，会进入下面的循环，在这里状态被改成了 executing，在循环中最终的结果就是返回了 value 和 done，只是在循环中，增加了运行状态的一些判定。

现在整片源码，我们剩下的只有循环中的数据处理，以及联系 context 上下文方法解读这两个部分没有分析完毕，我们再继续看这两个部分。

先看循环内部，循环的一开始，我们声明了 record 对象，定义了 type 为 normal，arg 为 innerFn 的返回值，也就是 switch case 那部分函数的返回值，而 innerFn 的返回值就是我们设定的每一步的结果。
再联系 helloWorldGenerator$，也就是 innerFn 的内部逻辑，只有在倒数第二步的时候才通过调用 context 中的 abrupt 修改了 record 的 type，把 type 修改为了 return，abrupt 又调用了 complete 方法，complete 方法把 record 里面的 arg，也就是我们设定的状态赋值给了 context 内部的 rval 和自己的 arg，然后返回了 ContinueSentinel 这个空对象（这也是为什么 record.arg === ContinueSentinel）。

这里我们连起来看，就是在没有执行到倒数第二步的时候，循环内声明的 record 的 type 一直是 normal，arg 一直是我们已经写好了的结果，到了倒数第二步的时候会有一些不同，这里我们的 type 变成了 return，arg 变成了 ContinueSentinel 这个空对象。然后在循环内部，record.arg === ContinueSentinel 这个判定生效，没有执行到 return，直接 continue 进入下一轮的循环。
而最后一轮的循环就很清晰了，调用了 stop 方法，stop 把 done 变成了 true，把我们在上一轮存起来的 arg 返回，形成了最终的结果，在最后一轮循环中，state 因为 context.done 的值发生了变化， context.done ? ‘completed’ : 'yield 的三元运算符也将取到 completed 的值，这样保证了在最后一次执行结束后，再进行调用的时候再函数的上层就直接 return 了{ value: undefined, done: true }这个结果，而且在 complete 调用时，也修改了 next 为 end，同时保证了在拿到最终结果多次调用的时候也会走 invoke 函数。

至此为止，对源码的分析已经结束了，但是我们回过头看，这个 invoke 的功能是不是觉得有几分熟悉？这不就是调用 generator 后，返回的对象的 next 方法吗？可是为什么变了个名字？我们再查阅源码可以看出，其实 invoke 就是 next 方法。

```javascript
// Helper for defining the .next, .throw, and .return methods of the
// Iterator interface in terms of a single ._invoke method.
function defineIteratorMethods(prototype) {
  ;['next', 'throw', 'return'].forEach(function (method) {
    prototype[method] = function (arg) {
      return this._invoke(method, arg)
    }
  })
}

defineIteratorMethods(Gp)
```

参考资料 1：https://blog.csdn.net/qq_46193451/article/details/110064977
参考资料 2：https://zhuanlan.zhihu.com/p/216060145
