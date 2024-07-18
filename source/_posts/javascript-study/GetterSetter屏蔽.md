---
title: JavaScript 中 Getter/Setter 属性访问/操作符的”屏蔽“作用
date: 2024-04-17 14:16:03
categories: 基础
tags: 基础
---

## 基本对象操作

首先我们先来看看一段非常普通的 JS 代码：

```javascript
const parent = {
  name: '19Qingfeng',
}

const child = Object.create(parent)

child.name = 'WangHaoyu'
console.log(child)
console.log(child.name)
```

相信这段代码对于大家来说都是信手拈来，我们通过 Object.create 方法创建了一个新的对象 child ，同时让 child 对象的 protot 指向了 parent 对象从而实现了继承的关系。  
当执行 `child.name = 'WangHaoyu'` 时，实质上相当于我们在为 child 实例对象上进行赋值操纵自然而然和原型上的 name 属性并不会有任何关系。  
同样当我们访问 child.name 时，因为实例本身存在 name 属性自然是不会去原型链上查找了。理所应当输出 WangHaoyu 。

## Getter/Setter

在 JavaScript 定义对象时，我们同时可以通过 [[Getter]]、[[Setter]] 来为属性绑定对应的执行函数。

```javascript
const obj = {
  _name: null,
  get name() {
    return this._name
  },
  set name(value) {
    this._name = value
  },
}

obj.name = '19Qingfeng'

console.log(obj.name) // 19Qingfeng
```

比如上边的示例中我们为 obj 对象定义了一个 get 属性访问符 name ，同时也为 name 定义了对应的 setter 属性操作方法。  
当执行 `obj.name = '19Qingfeng'` 时，实际上是会调用 obj 上的名为 name 的 setter 函数，从而修改 obj 实例对象上的 \_name 值。

## 屏蔽效果

```javascript
// 创建parent对象 拥有get/set以及_name
const parent = {
  _name: null,
  get name() {
    return this._name
  },
  set name(value) {
    this._name = value
  },
}

// 创建一个空对象child 相当于 child.__proto__ = parent 实现原型继承
const child = Object.create(parent)

// 为child实例赋值name属性
child.name = '19Qingfeng'

console.log(child, 'child')
```

大家可以稍微想想这里的操作，其实它并不难。我们通过 Object.create 创建了一个空对象，同时让他继承与 parent 对象。  
**在之后我们通过 `child.name = '19Qingfeng'` 尝试为 child 实例添加一个 name 属性值为 19Qingfeng。**  
此时如果按照我们基于 JavaScript 的理解的话，我们为 child 实例上添加了一个 name 属性，实质上是和原型上的 parent 中的同名操作符没有任何关系对吧。  
最开始我也是天真的这样以为的，当我们进行 `child.name = '19Qingfeng'` 赋值时，应该仅仅为 child 实例上添加一个 name 为 19Qingfeng 的属性就可以了。  
可是结果并不是这样  
![屏蔽](https://note.youdao.com/yws/api/personal/file/WEBc07227ad3b9dd70d8e628324a3eaafe4?method=download&shareKey=dcb7af32618acde118fc337acee8cfc1)  
**我们明明是在实例 child 上进行了赋值，可是为什么 child 上并没有出现所谓的 name 属性，而是拥有了一个名为 \_name 的 19Qinfeng ？**  
其实这正是我想和大家重点强调的的所谓 Getter/Setter 产生的屏蔽效应：  
比如上边我们为 child 的 name 属性进行赋值操作时完整过程如下：

- 如果 child 对象中包含名为 name 的普通数据访问属性，那么这条赋值语句机会修改已有的属性值。
- 如果 name 属性并不是存在于 child 实例对象上，那么此时会遍历 child 的 prototype 查找这个属性。
- 如果原型链中查找不到这个 name 属性，那么很简单此时 name 属性会被直接添加到 child 实例对象上。

**但是，如果 child 的原型链中查询到了 name 属性，那么此时情况就会稍微有些复杂。**

- 第一种即为我们熟知的，如果属性 name 同时出现在了实例上以及原型 prototype 中的话，那么此时首先就会发生所谓的屏蔽。

对于 child 中的 name 属性的修改，会屏蔽原型上所有 name 属性的修改，这也是我们的理解。

当然如果屏蔽真的如此简单的话，那么我完全没有必要和大家来强调它，**如果实例中并不直接存在 name 属性但是此时原型上存在同名的 name 时**

- 首先第一情况下，如果 child 的原型链 prototype 上存在名为 name 的普通数据访问属性，并且此时该属性没有被标记为只读（writable:false）,**那么此时会在 child 实例上添加一个 name 属性，它会屏蔽原型上的属性。**
- 其次第二种情况下，如果 child 的原型链 prototype 上存在名为 name 的普通数据访问属性，并且该属性被标记为只读(writable:true)，**那么此时对于 `child.name = '19Qingfeng'` 是不会产生任何效果的，换句话说它并不会为自身实例添加属性同时也无法修改原型上的同名属性，在严格模式下甚至这一行为会提示错误。**
- 最后第三种情况下，它可以解释我们刚刚的行为。如果 child 的原型链上存在一个 name 并且此时他是一个 setter 时，**那么此时我们在实例上进行赋值操作时，原型上的同名 setter 会被调用，并且 name 属性并不会被添加到实例中，同时也不会对原型上的 setter 造成任何影响。**

结合第三种情况下，我们来解释刚才的例子。首先，child 对象中本身并不存在 name 属性，但是它继承与 parent 对象。child 的原型上存在所谓的名为 name 的 getter 和 setter 。  
当我们调用 `child.name = '19QIngfeng'` 时，会满足屏蔽效果下的第三种情况：  
针对于实例上的 `child.name = '19QIngfeng'` 并不会为实例添加属性，并且会调用原型上最近的 setter 操作，此时相当于会执行 `this._name = value` 。  
因为我们通过 `child.name` 调用，所以 setter 里的 this 会指向对应的 `child` ，自然 setter 中的逻辑就相当于为 child 实例添加了一个普通属性 \_name 值为 19QIngfeng 。

https://juejin.cn/post/7074935443355074567
