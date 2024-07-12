---
title: 为什么Proxy一定要配合Reflect使用？
date: 2024-04-17 13:24:41
categories: 基础
tags: 基础
---

## 单独使用 Proxy

```javascript
const obj = {
  name: 'wang.haoyu',
}

const proxy = new Proxy(obj, {
  // get陷阱中target表示原对象 key表示访问的属性名
  get(target, key) {
    console.log('劫持你的数据访问' + key)
    return target[key]
  },
})

proxy.name // 劫持你的数据访问name -> wang.haoyu
```

看起来很简单对吧，我们通过 Proxy 创建了一个基于 obj 对象的代理，同时在 Proxy 中声明了一个 get 陷阱。  
当访问我们访问 proxy.name 时实际触发了对应的 get 陷阱，它会执行 get 陷阱中的逻辑，同时会执行对应陷阱中的逻辑，最终返回对应的 `target[key]` 也就是所谓的 `wang.haoyu` .

## Proxy 中的 receiver

上边的 Demo 中一切都看起来顺风顺水没错吧，细心的同学在阅读 Proxy 的 MDN 文档上可能会发现其实 Proxy 中 get 陷阱中还会存在一个额外的参数 receiver 。  
那么这里的 receiver 究竟表示什么意思呢？**大多数同学会将它理解成为代理对象，但这是不全面的**。  
接下来同样让我们以一个简单的例子来作为切入点：

```javascript
const obj = {
  name: 'wang.haoyu',
}

const proxy = new Proxy(obj, {
  // get陷阱中target表示原对象 key表示访问的属性名
  get(target, key, receiver) {
    console.log(receiver === proxy)
    return target[key]
  },
})

// log: true
proxy.name
```

上述的例子中，我们在 Proxy 实例对象的 get 陷阱上接收了 receiver 这个参数。  
同时，我们在陷阱内部打印 `console.log(receiver === proxy)`; 它会打印出 true ，表示这里 receiver 的确是和代理对象相等的。  
**所以 receiver 的确是可以表示代理对象，但是这仅仅是 receiver 代表的一种情况而已。**  
接下来我们来看另外一个例子：

```javascript
const parent = {
  get value() {
    return '19Qingfeng'
  },
}

const proxy = new Proxy(parent, {
  // get陷阱中target表示原对象 key表示访问的属性名
  get(target, key, receiver) {
    console.log(receiver === proxy)
    return target[key]
  },
})

const obj = {
  name: 'wang.haoyu',
}

// 设置obj继承与parent的代理对象proxy
Object.setPrototypeOf(obj, proxy)

// log: false
obj.value
```

我们可以看到，上述的代码同样我在 proxy 对象的 get 陷阱上打印了 `console.log(receiver === proxy);` 结果却是 false 。  
那么你可以稍微思考下这里的 receiver 究竟是什么呢？ 其实这也是 proxy 中 get 陷阱第三个 receiver 存在的意义。  
**它是为了传递正确的调用者指向，你可以看看下方的代码：**

```javascript
...
const proxy = new Proxy(parent, {
  // get陷阱中target表示原对象 key表示访问的属性名
  get(target, key, receiver) {
-   console.log(receiver === proxy) // log:false
+   console.log(receiver === obj) // log:true
    return target[key];
  },
});
...

```

**其实简单来说，get 陷阱中的 receiver 存在的意义就是为了正确的在陷阱中传递上下文。**  
涉及到属性访问时，不要忘记 get 陷阱还会触发对应的属性访问器，也就是所谓的 get 访问器方法。  
我们可以清楚的看到上述的 receiver 代表的是继承与 Proxy 的对象，也就是 obj。  
**看到这里，我们明白了 Proxy 中 get 陷阱的 receiver 不仅仅代表的是 Proxy 代理对象本身，同时也许他会代表继承 Proxy 的那个对象。**  
其实本质上来说它还是为了确保陷阱函数中调用者的正确的上下文访问，比如这里的 receiver 指向的是 obj 。

> 当然，你不要将 revceiver 和 get 陷阱中的 this 弄混了，陷阱中的 this 关键字表示的是代理的 handler 对象。

```javascript
const parent = {
  get value() {
    return '19Qingfeng'
  },
}

const handler = {
  get(target, key, receiver) {
    console.log(this === handler) // log: true
    console.log(receiver === obj) // log: true
    return target[key]
  },
}

const proxy = new Proxy(parent, handler)

const obj = {
  name: 'wang.haoyu',
}

// 设置obj继承与parent的代理对象proxy
Object.setPrototypeOf(obj, proxy)

// log: false
obj.value
```

## Reflect 中的 receiver

在清楚了 Proxy 中 get 陷阱的 receiver 后，趁热打铁我们来聊聊 Reflect 反射 API 中 get 陷阱的 receiver。  
我们知道在 Proxy 中（以下我们都以 get 陷阱为例）第三个参数 receiver 代表的是代理对象本身或者继承与代理对象的对象，它表示触发陷阱时正确的上下文。

```javascript
const parent = {
  name: '19Qingfeng',
  get value() {
    return this.name
  },
}

const handler = {
  get(target, key, receiver) {
    return Reflect.get(target, key)
    // 这里相当于 return target[key]
  },
}

const proxy = new Proxy(parent, handler)

const obj = {
  name: 'wang.haoyu',
}

// 设置obj继承与parent的代理对象proxy
Object.setPrototypeOf(obj, proxy)

// log: false
console.log(obj.value)
```

我们稍微分析下上边的代码：

- 当我们调用 obj.value 时，由于 obj 本身不存在 value 属性。
- 它继承的 proxy 对象中存在 value 的属性访问操作符，所以会发生屏蔽效果。
- 此时会触发 proxy 上的 get value() 属性访问操作。
- 同时由于访问了 proxy 上的 value 属性访问器，所以此时会触发 get 陷阱。
- 进入陷阱时，target 为源对象也就是 parent ，key 为 value 。
- 陷阱中返回 `Reflect.get(target,key)` 相当于 `target[key]`。
- 此时，不知不觉中 this 指向在 get 陷阱中被偷偷修改掉了！
- 原本调用方的 obj 在陷阱中被修改成为了对应的 target 也就是 parent 。
- 自然而然打印出了对应的 `parent[value]` 也就是 19Qingfeng 。

这显然不是我们期望的结果，当我访问 obj.value 时，我希望应该正确输出对应的自身上的 name 属性也就是所谓的 obj.value => wang.haoyu 。  
那么，Relfect 中 get 陷阱的 receiver 就大显神通了。

```javascript
const parent = {
  name: '19Qingfeng',
  get value() {
    return this.name;
  },
};

const handler = {
  get(target, key, receiver) {
-   return Reflect.get(target, key);
+   return Reflect.get(target, key, receiver);
  },
};

const proxy = new Proxy(parent, handler);

const obj = {
  name: 'wang.haoyu',
};

// 设置obj继承与parent的代理对象proxy
Object.setPrototypeOf(obj, proxy);

// log: wang.haoyu
console.log(obj.value);

```

上述代码原理其实非常简单：

> 你可以简单的将 Reflect.get(target, key, receiver) 理解成为 target[key].call(receiver)，不过这是一段伪代码，但是这样你可能更好理解。

https://juejin.cn/post/7080916820353351688
