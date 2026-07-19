---
layout:     post
title:      写给前端的 Generator 指南：从 yield 到 async/await 的完整进化
subtitle:   写给前端的 Generator 指南：从 yield 到 async/await 的完整进化
date:       2026-07-16
author:     LBX
header-img: img/bg-little-universe.jpg
catalog: true
tags:
    - 前端
    - javascript
    - promise
    - 微任务
    - 宏任务
    - generator
---

> 概念、yield/next/throw/return、Iterator 协议

## 写在前面

你每天都在写 async/await，但有没有想过——JS 引擎是怎么做到「暂停执行，等异步回来再继续」的？

这个问题，和 Generator 函数有关。

在 JS 异步编程的演进链路上——回调 → Promise → **Generator** → async/await——Generator 是承前启后的一环。它第一次让异步代码拥有了同步写法的可读性。后来 async/await 借鉴了这个思路，把「同步风格写异步」内置到了语言层面。

需要说明的是，async/await 在引擎层面有自己独立的实现（基于 Promise 微任务），并不是 Generator 的语法糖。但 Generator 是这条演进链路上理解「为什么异步代码能写成同步风格」的关键一环，了解了它，你会更清楚 async/await 在解决什么问题。

这系列拆成三篇：
- **第一篇**（本文）：Generator 概念、核心语法、Iterator 协议
- **第二篇**：Babel 编译产物剖析、V8 引擎层面的暂停机制
- **第三篇**：自动执行器、async/await 与 Generator 的关系、高阶玩法 + 面试题

---

## 一、Generator 是什么？

### 1.1 一句话概念

> **Generator 函数是一个可以「暂停」和「恢复」执行的函数。**

普通函数一旦调用，从头跑到尾，中间不停。Generator 不一样——它可以执行到一半暂停，把控制权交出去，等外面告诉它继续，再从暂停的地方接着跑。

### 1.2 看个例子

```js
function* helloGenerator() {
  yield 'hello';
  yield 'world';
  return 'ending';
}

const g = helloGenerator();

g.next(); // { value: 'hello', done: false }
g.next(); // { value: 'world', done: false }
g.next(); // { value: 'ending', done: true }
g.next(); // { value: undefined, done: true }
```

和普通函数相比，就两个区别：`function` 后面多了个星号 `*`，函数体内用 `yield` 关键字标记「暂停点」。

有个细节容易忽略——**调用 Generator 函数不会立刻执行函数体**，而是返回一个遍历器对象（Iterator）。调用这个对象的 `.next()` 方法，函数体才会开始执行，执行到 `yield` 处暂停。

这和普通函数完全不同。普通函数是「调用即执行」，Generator 是「调用只拿到句柄，next 才真正跑」。

### 1.3 协程：Generator 的前世

要真正理解 Generator 为什么能暂停，得先聊一个概念——协程（Coroutine）。

普通函数是「子例程」（Subroutine），调用关系是单向的：A 调用 B，B 执行完返回 A，中间不能「暂停「切换回去」。协程不一样，它可以随时挂起自己，把控制权还给调用者，等调用者再次唤起它时从挂起的位置继续。

```
普通函数：  A → B → B 执行完 → A 继续
协程：      A → B → B 挂起 → A 继续 → A 唤起 B → B 从挂起处继续
```

很多人容易把协程和线程搞混。简单来说——

**线程是操作系统调度的，协程是用户态调度的。**

通俗一点来说：线程是操作系统帮你切换，你管不了什么时候切，所以要加锁防冲突；协程是你自己（或者说代码）主动让出控制权，什么时候切你说了算，不存在抢夺，自然不用加锁。

JS 是单线程的，所以它走的路线就是协程这条路——Generator 的 `yield` 就是「主动让出控制权」，`next()` 就是「主动拿回控制权」。没有操作系统的参与，没有线程切换的开销。

> **线程 vs 协程调度对比图** — 左右对比两种调度方式的执行时间线和核心差异

Generator 就是 JS 对协程的轻量实现。`yield` 是挂起点，`next()` 是恢复信号。虽然 JS 的 Generator 不是完整的协程（不能从任意位置恢复，只能在 yield 处），但核心思想完全一致。

理解了这一层，你就明白为什么 Generator 能做异步编程了——它可以在异步操作开始时挂起，等异步完成后再从外部 `next()` 恢复执行。这正是 async/await 同步写法的灵感来源。

---

## 二、为什么需要 Generator？

### 2.1 异步演进的痛点

JS 异步编程的演进路线是这样的：

**回调 → Promise → Generator → async/await**

每一步都在解决上一步的痛点。回调地狱就不用再展开了，写过的人都懂。Promise 把嵌套改成了链式，好多了，但写复杂逻辑还是很别扭：

```js
// Promise 版本：条件分支 + 循环，写起来很别扭
function processItems(items) {
  return items.reduce(
    (promise, item) =>
      promise.then(results =>
        fetchDetail(item.id).then(detail => {
          if (detail.type === 'special') {
            return fetchExtra(detail.id).then(extra => {
              results.push({ ...detail, extra });
              return results;
            });
          }
          results.push(detail);
          return results;
        })
      ),
    Promise.resolve([])
  );
}
```

逻辑不复杂——遍历 items，逐个请求详情，遇到 special 类型就额外请求一次。但用 Promise 写出来，嵌套和 `.then` 的层级让你得多读两遍才能看明白它在干嘛。

### 2.2 Generator 的定位

Generator 的价值在于——它让异步代码可以写成同步风格：

```js
function* processItems(items) {
  const results = [];
  for (const item of items) {
    const detail = yield fetchDetail(item.id);
    if (detail.type === 'special') {
      const extra = yield fetchExtra(detail.id);
      results.push({ ...detail, extra });
    } else {
      results.push(detail);
    }
  }
  return results;
}
```

同样的逻辑，Generator 版本读起来和同步代码几乎一样——`for` 循环、`if` 判断、变量赋值，该怎么写就怎么写。唯一的区别是异步操作前面加了 `yield`。

Generator 在异步演进中的定位就是这个：**它第一次让 JS 开发者用同步写法写异步逻辑。** 后来 async/await 借鉴了这个思路，把同步风格写异步的能力内置到了语言层面。不过两者在引擎层面的实现是独立的——async/await 基于 Promise 微任务机制实现，并非直接基于 Generator，但 Generator 是这条演进链路上不可或缺的一环。

---

## 三、Generator 核心语法

### 3.1 next()：启动和恢复

`.next()` 是控制 Generator 执行的基本方法。每次调用，Generator 从当前暂停处恢复执行，直到遇到下一个 `yield` 或 `return`，然后再次暂停。

```js
function* gen() {
  console.log('开始执行');
  yield 1;
  console.log('第一次恢复');
  yield 2;
  console.log('第二次恢复');
  return 3;
}

const g = gen();
g.next();
// 打印：开始执行
// 返回：{ value: 1, done: false }

g.next();
// 打印：第一次恢复
// 返回：{ value: 2, done: false }

g.next();
// 打印：第二次恢复
// 返回：{ value: 3, done: true }

g.next();
// 不打印任何东西
// 返回：{ value: undefined, done: true }
```

几个要点：
- 第一次 `next()` 启动 Generator，执行到第一个 `yield` 暂停
- `yield` 后面的表达式值会被包装成 `{ value, done: false }` 返回
- 遇到 `return`，`done` 变成 `true`
- `done: true` 之后再调 `next()`，永远返回 `{ value: undefined, done: true }`

> **Generator 执行流程图** — 展示 next() 调用如何驱动 Generator 从 yield 到 return 的完整流程

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/058fc3927232438f88c58c70e67ad6bc.png#pic_center)


### 3.2 yield 的双向通信

`yield` 不只是往外产出值，还能接收外面传进来的值。这是 Generator 做异步编程的关键。

```js
function* gen() {
  const a = yield 1;
  const b = yield a + 1;
  return a + b;
}

const g = gen();
g.next();     // { value: 1, done: false } —— 执行到 yield 1，暂停
g.next(10);   // { value: 11, done: false } —— 10 赋给 a，执行到 yield 11
g.next(20);   // { value: 30, done: true } —— 20 赋给 b，return 10+20
```

这里有个反直觉的点：`next(10)` 的参数 `10` 不是传给「下一个 yield」，而是赋给**上一次暂停的那个 yield 表达式**。

也就是说，`const a = yield 1` 这行代码被拆成了两步：
1. 第一次 `next()`：执行到 `yield 1`，产出 `1`，暂停
2. 第二次 `next(10)`：`10` 作为 `yield 1` 整体的返回值赋给 `a`，继续执行，可以理解为`const a = 10`

**第一个 `next()` 的参数永远无效**——因为第一个 yield 之前没有「上一个 yield」来接收它。这是个高频踩坑点，记住这个心智模型：

```
next()      → 启动/恢复，执行到下一个 yield，产出值
next(value) → value 填入「刚才暂停处的 yield」，继续执行到下一个 yield
```

说实话，我第一次学 Generator 的时候在这个地方卡了很久——直觉上总以为 `next(10)` 的 `10` 是传给下一个 yield 的。后来才想明白，这个设计确实不直观，但理解了之后你会觉得它其实很合理。

**来个更实际的例子**——用 yield 双向通信模拟一个交互式问答流程：

```js
function* interview() {
  const name = yield '你叫什么名字？';
  const years = yield `你好 ${name}，你有几年经验？';
  const skill = yield `收到，${years} 年经验。你最擅长什么技术？`;
  return `面试结束：${name}，${years}年经验，擅长${skill}`;
}

const it = interview();
console.log(it.next().value);            // 你叫什么名字？
console.log(it.next('蜡笔熊').value);     // 你好 蜡笔熊，你有几年经验？
console.log(it.next('5').value);          // 收到，5 年经验。你最擅长什么技术？
console.log(it.next('前端工程化').value);  // 面试结束：蜡笔熊，5年经验，擅长前端工程化
```

Generator 像一个可以「暂停等待输入」的状态机。每次 `next(value)` 把外面的数据传进去，Generator 处理后再 `yield` 出新的问题。Redux Saga 里的交互逻辑就是这么实现的。

> yield 双向通信数据流图** — 展示 next(value) 参数如何传入、yield 产出值如何传出，以及「参数赋给上一个 yield」的核心机制

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/260ddb15012944deb6e63fe6050f531a.png#pic_center)


### 3.3 yield*：委托给另一个 Generator

`yield*` 后面跟一个可迭代对象（另一个 Generator、数组、字符串等），会把它们的值逐个产出：

```js
function* inner() {
  yield 'a';
  yield 'b';
  return 'inner-result';
}

function* outer() {
  yield 1;
  yield* inner();
  yield 2;
}

const g = outer();
g.next(); // { value: 1, done: false }
g.next(); // { value: 'a', done: false }
g.next(); // { value: 'b', done: false }
g.next(); // { value: 2, done: false }
g.next(); // { value: undefined, done: true }
```

`yield*` 和手动遍历有个关键区别：`yield*` 能拿到内层 Generator 的 `return` 值。

```js
function* outer() {
  const result = yield* inner();
  console.log(result); // 'inner-result'
}
```

而 `for...of` 遍历 Generator 时会忽略 `return` 值——这点后面踩坑篇会细说。

### 3.4 throw()：从外部抛入错误

`.throw(error)` 会在**上次暂停的 yield 表达式处**抛出一个错误。如果 Generator 内部有 `try/catch`，可以捕获它并继续执行：

```js
function* gen() {
  try {
    yield '正常执行';
  } catch (e) {
    console.log('内部捕获:', e.message);
    yield '错误后恢复';
  }
  return '结束';
}

const g = gen();
g.next();                       // { value: '正常执行', done: false }
g.throw(new Error('出错了'));   // 打印：内部捕获: 出错了
                                // → { value: '错误后恢复', done: false }
g.next();                       // { value: '结束', done: true }
```

如果内部没有 `try/catch`，错误会穿透到外部，Generator 直接终止：

```js
function* gen() {
  yield 1;
  yield 2;
}

const g = gen();
g.next();  // { value: 1, done: false }

try {
  g.throw(new Error('没人接'));
} catch (e) {
  console.log('外部捕获:', e.message);  // 外部捕获: 没人接
}

// Generator 已终止，后续 next() 只返回 { value: undefined, done: true }
g.next();  // { value: undefined, done: true }
```

### 3.5 return()：强制终止

`.return(value)` 立即终止 Generator，返回 `{ value, done: true }`：

```js
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}

const g = gen();
g.next();              // { value: 1, done: false }
g.return('强制结束');  // { value: '强制结束', done: true }
g.next();              // { value: undefined, done: true }
```

如果 Generator 内部有 `try...finally`，`return()` 会先执行 `finally` 块：

```js
function* gen() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log('finally 执行了');  // ← return() 触发了它
  }
}

const g = gen();
g.next();             // { value: 1, done: false }
g.return('结束');     // 打印：finally 执行了
                      // → { value: '结束', done: true }
```

这三个方法（`next`、`throw`、`return`）构成了 Generator 的外部控制接口。普通函数你只能调用它，不能从外部干预它的执行流程。而 Generator 把「暂停/恢复/报错/终止」的控制权都交给了外部，也是后面理解自动执行器和 Redux Saga 的基础。

### 3.6 方法速查表

| 方法 | 作用 |
|------|------|
| `next()` | 启动或恢复执行，返回 `{ value, done }` |
| `next(value)` | 恢复执行，value 作为上一个 yield 的返回值 |
| `throw(error)` | 在上次暂停的 yield 处抛出错误 |
| `return(value)` | 强制终止 Generator 并返回值 |
| `yield*` | 委托另一个 Generator 或可迭代对象 |

---

## 四、Generator 与 Iterator 协议

> 为什么有些对象能用 `for...of` 遍历，有些不能？答案就在这里。

### 4.1 Iterator 是什么？

Generator 函数调用后返回的对象，天然就是一个 Iterator（遍历器）。要理解这层关系，得先知道 Iterator 是什么。

> **Generator 与 Iterator 协议关系图** — 三层关系：Iterable → Iterator → Generator，以及常用语法如何调用 Symbol.iterator

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1110c86bfc1f424a8f40194bb280e84f.png#pic_center)


JS 有很多数据结构需要遍历：Array、Map、Set、String、NodeList……如果每种结构都有一套自己的遍历方式，代码会很混乱。Iterator 的作用就是**为所有数据结构提供统一的访问机制**。

Iterator 的本质是一个指针对象，定义了 `next()` 方法，每次调用返回 `{ value, done }`：

```js
// 手写一个最简单的 Iterator
function makeIterator(array) {
  let index = 0;
  return {
    next() {
      return index < array.length
        ? { value: array[index++], done: false }
        : { value: undefined, done: true };
    }
  };
}

const it = makeIterator(['a', 'b']);
it.next();  // { value: 'a', done: false }
it.next();  // { value: 'b', done: false }
it.next();  // { value: undefined, done: true }
```

**Iterator 不一定需要对应真实的数据结构**。你可以写一个无限运行的 Iterator：

```js
function idMaker() {
  let index = 0;
  return {
    next() {
      return { value: index++, done: false };
    }
  };
}

const ids = idMaker();
ids.next().value;  // 0
ids.next().value;  // 1
ids.next().value;  // 2
// ...无限产出
```

这个特性跟Generator特别契合，Generator 本质上就是一个自带 next 方法的 Iterator 生成器。

### 4.2 Symbol.iterator：可遍历协议

ES6 规定，一个数据结构只要部署了 `Symbol.iterator` 属性，就被视为可遍历的（iterable）。`for...of`、展开运算符 `...`、解构赋值等语法在内部都会自动调用这个属性。

原生具备 Iterator 接口的数据结构：Array、Map、Set、String、TypedArray、arguments、NodeList。

普通对象（Object）没有——因为对象的属性遍历顺序不确定，需要开发者手动指定。

```js
let arr = ['a', 'b', 'c'];
let iter = arr[Symbol.iterator]();

iter.next();  // { value: 'a', done: false }
iter.next();  // { value: 'b', done: false }
iter.next();  // { value: 'c', done: false }
iter.next();  // { value: undefined, done: true }
```

你可能不知不觉已经在用 Iterator 了——这些场景都会默认调用 `Symbol.iterator`：

```js
// 解构赋值
const [x, y] = new Set([1, 2]);  // 调用 Set 的 Iterator

// 扩展运算符
const arr = [...'hello'];  // ['h', 'e', 'l', 'l', 'o']

// yield*
yield* [2, 3, 4];

// Promise.all 接受任何可迭代对象
Promise.all(new Set([p1, p2, p3]));
```

任何接受可迭代对象的 API，内部都在调用 `Symbol.iterator`。只要部署了这个接口，就可以用 `for...of`、`...`、解构等所有语法糖。

### 4.3 用 Generator 实现 Iterator 接口

手动写 Iterator 的 `next()` 方法很繁琐。用 Generator，只需要写 `yield` 语句：

```js
// 让普通对象可被 for...of 遍历
const obj = { a: 1, b: 2, c: 3 };

obj[Symbol.iterator] = function* () {
  for (const key of Object.keys(this)) {
    yield [key, this[key]];
  }
};

for (const [k, v] of obj) {
  console.log(k, v);  // a 1 → b 2 → c 3
}

// 现在也可以用展开运算符了
console.log([...obj]);  // [['a', 1], ['b', 2], ['c', 3]]
```

Generator 函数调用后返回的对象，天然实现了 `next()` 方法，天然返回 `{ value: xxx, done: true/false }`——天生就是一个 Iterator。这就是为什么用 Generator 来实现 `Symbol.iterator` 是最简洁的方式。

---

## 五、基础应用场景

### 场景一：异步流程的同步化表达

```js
function* loadPage() {
  showLoading();
  const data = yield fetch('/api/data');   // 暂停等请求
  render(data);
  hideLoading();
}

// 手动执行
const loader = loadPage();
loader.next().value.then(response => {
  loader.next(response);
});
```

这是 Generator 最经典的用法——把异步操作写成同步风格。当然，手动执行很痛苦（每次都要调 `next`），第二篇会讲怎么用执行器自动化。而这正是 async/await 设计思路的来源。

### 场景二：惰性序列

Generator 天然适合生成无限序列，因为它是「按需产出」的：

```js
function* naturalNumbers() {
  let n = 1;
  while (true) {
    yield n++;
  }
}

const numbers = naturalNumbers();
numbers.next().value;  // 1
numbers.next().value;  // 2
numbers.next().value;  // 3
// ...永远不会爆内存
```

`while(true)` 在普通函数里是个灾难，但在 Generator 里完全没问题——因为只有调用 `next()` 才会执行一步。这就是惰性求值的威力。

### 场景三：状态机

```js
function* trafficLight() {
  while (true) {
    yield '🟢 绿灯';
    yield '🟡 黄灯';
    yield '🔴 红灯';
  }
}

const light = trafficLight();
setInterval(() => {
  console.log(light.next().value);
}, 3000);
// 每 3 秒切换一次信号灯
```

用 Generator 实现状态机，不需要外部变量来保存状态——状态就内嵌在函数的执行位置里。执行到哪个 yield，就是哪个状态。

---

## 本篇小结

| 要点 | 说明 |
|------|------|
| `function*` | 声明 Generator 函数 |
| `yield` | 暂停执行并产出值 |
| `.next(value)` | 恢复执行，value 作为上一个 yield 的返回值 |
| `.throw(error)` | 向 Generator 内部抛出错误 |
| `.return(value)` | 终止 Generator 并返回值 |
| `yield*` | 委托另一个 Generator 或可迭代对象 |
| `Symbol.iterator` | 可遍历协议，Generator 天然实现这个接口 |

Generator 的核心就这些——`function*` 声明、`yield` 暂停、`next` 恢复、`throw/return` 外部控制。语法本身不复杂，但它的执行机制和普通函数完全不同。

那么**JS 引擎到底是怎么实现「暂停」的？** 一个函数执行到一半挂起，它的调用栈、局部变量、执行上下文是怎么保存的？`yield` 在底层到底发生了什么？

下一篇我们会把 Generator 放到 Babel 编译器下，看看 ES6 的 `function*` 被编译成 ES5 后是什么样子——你会发现，所谓的「暂停」和「恢复」，本质上是一个**状态机**。答案可能出乎你的意料。

**系列预告：**
- **第二篇**：Babel 编译产物剖析 + V8 引擎层面的暂停机制
- **第三篇**：自动执行器 + async/await 与 Generator 的关系 + 高阶玩法 + 面试题（系列收尾）

---

*如果觉得有帮助，欢迎关注。*
