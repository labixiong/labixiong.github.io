---
layout:     post
title:      实现一个能跑的迷你版Promise（一）
subtitle:   手写Promise第一篇，实现一个能跑的迷你版Promise（一）
date:       2026-06-29
author:     LBX
header-img: img/bg-little-universe.jpg
catalog: true
tags:
    - 前端
    - javascript
    - promise
    - 微任务
    - 宏任务
---

# 微任务、静态方法、async/await 全搞懂（三）

## 回顾

回顾一下前两篇的成果：

- **第一篇**：搭出了一个 50 行的 mini Promise，能跑异步、有 catch/finally
- **第二篇**：实现了链式调用 + resolvePromise 解析过程，通过了 Promise/A+ 全套 872 个测试

现在是最后一步：**把 Promise 放进 JavaScript 运行时的大图景里**，下面是文章具体要讲述的内容：

1. 为什么 `Promise.then` 的回调比 `setTimeout` 先执行？微任务 —— Promise 回调为什么慢一步
2. `Promise.all/race/allSettled/any` 内部怎么实现？实现全部静态方法
3. `async/await` 跟 Promise 和 Generator 到底什么关系？async/await的本质
4. 实际工作中，手写 Promise 的能力有什么实战价值？Promise并发控制
4. 手写Promise的价值

---

## 第一章：微任务 —— Promise 回调为什么慢一步

### 1.1 经典面试题分布拆解

```js
console.log('1: 同步');

setTimeout(() => {
  console.log('2: setTimeout');
}, 0);

new Promise((resolve) => {
  console.log('3: Promise 构造函数');
  resolve('4: then 回调');
}).then((value) => {
  console.log(value);
});

console.log('5: 同步');
```

**输出顺序是：** `1 → 3 → 5 → 4 → 2`

一句句分析执行过程：

```
第 1 步：console.log('1: 同步')
  调用栈：↓ 立刻执行 ↓
  输出：1: 同步

第 2 步：setTimeout(..., 0)
  调用栈：把回调注册到 Web API 的定时器
  Web API 在 0ms 后把回调塞进「宏任务队列」
  宏任务队列：[ () => console.log('2: setTimeout') ]

第 3 步：new Promise(executor)
  executor 同步执行 → console.log('3: Promise 构造函数')
  输出：3: Promise 构造函数
  resolve('4: then 回调') → 状态变 fulfilled
  .then(callback) → 回调被塞进「微任务队列」
  微任务队列：[ () => console.log('4: then 回调') ]

第 4 步：console.log('5: 同步')
  输出：5: 同步
  调用栈清空！

--- 当前宏任务（整体 script）执行完毕 ---
--- 检查微任务队列 ---

第 5 步：执行微任务队列
  () => console.log('4: then 回调')
  输出：4: then 回调
  微任务队列清空！

--- 微任务全部执行完毕 ---
--- 取出下一个宏任务 ---

第 6 步：执行宏任务队列
  () => console.log('2: setTimeout')
  输出：2: setTimeout
```

### 1.2 事件循环的完整流程

画成图就是这样的：

```
┌─────────────────────────────────────────────┐
│               一次事件循环 (Event Loop)        │
│                                               │
│  1. 取一个宏任务执行                            │
│     ├─ 整个 <script> 就是一个宏任务              │
│     ├─ setTimeout/setInterval 回调              │
│     └─ I/O、UI 渲染事件                         │
│                    ↓                           │
│  2. 执行「所有」微任务                           │
│     ├─ Promise.then/catch/finally 回调          │
│     ├─ MutationObserver 回调                    │
│     ├─ queueMicrotask 回调                     │
│     └─ process.nextTick (Node.js)              │
│                    ↓                           │
│  3. 渲染（浏览器）                               │
│     └─ 如果需要的话                              │
│                    ↓                           │
│  4. 回到第 1 步                                 │
└─────────────────────────────────────────────┘
```

**每个宏任务执行完毕后，必须清空微任务队列，才轮到下一个宏任务。**

### 1.3 微任务里产生新的微任务

```js
console.log('start');

Promise.resolve()
  .then(() => {
    console.log('微任务 1');
    // 在微任务里创建新微任务！
    Promise.resolve().then(() => {
      console.log('微任务 2');
    });
  })
  .then(() => {
    console.log('微任务 3');
  });

setTimeout(() => console.log('宏任务'), 0);

console.log('end');
```

**自己先猜一下输出顺序，再往下看：**


```
start
end
微任务 1
微任务 2    ← 被「插队」了！
微任务 3
宏任务
```

关键在于：**微任务执行过程中产生的新微任务，会在当前微任务队列清空阶段一并执行**，不会等到下一次事件循环。这就是「微任务可以插队」的含义。

### 1.4 把我们的 Promise 改用 queueMicrotask

第二篇用 `setTimeout` 模拟异步，但它本质上是宏任务，跟真正的 Promise 行为有差异。现在改一下：

```js
// 工具函数：用真正的微任务
function runMicrotask(fn) {
  if (typeof queueMicrotask === 'function') {
    // Node.js 11+、现代浏览器
    queueMicrotask(fn);
  } else if (typeof process !== 'undefined' && process.nextTick) {
    // 旧版 Node.js
    process.nextTick(fn);
  } else if (typeof MutationObserver !== 'undefined') {
    // 旧版浏览器 兜底方案
    const observer = new MutationObserver(fn);
    const node = document.createTextNode('');
    observer.observe(node, { characterData: true });
    node.data = '1';
  } else {
    // 终极兜底
    setTimeout(fn, 0);
  }
}
```

然后把 `then` 里的 `setTimeout` 全部换成 `runMicrotask`：

```js
// then 内部，把 setTimeout(..., 0) 替换为 runMicrotask(...)
if (this.state === FULFILLED) {
  runMicrotask(() => {
    try {
      const x = onFulfilled(this.value);
      resolvePromise(promise2, x, resolve, reject);
    } catch (e) {
      reject(e);
    }
  });
}
// ... 其他分支同理
```

### 对比 setTimeout 和 queueMicrotask 的差异

```js
// 用 setTimeout 模拟的版本
setTimeout(() => console.log('setTimeout 1'), 0);
myPromise.then(() => console.log('myPromise then'));
setTimeout(() => console.log('setTimeout 2'), 0);

// 如果用 setTimeout 实现：顺序不一定对
// 如果用 queueMicrotask 实现：myPromise then 一定在两个 setTimeout 之间
```

---

## 第二章：实现全部静态方法

### 2.1 Promise.resolve()

```js
MyPromise.resolve = function (value) {
  // 如果已经是 MyPromise 实例，直接返回
  if (value instanceof MyPromise) return value;

  // 创建新 Promise，用 resolvePromise 处理可能的 thenable
  return new MyPromise((resolve, reject) => {
    resolvePromise(null, value, resolve, reject);
  });
};
```

为什么要用 `resolvePromise` 而不是直接 `resolve(value)`？

```js
const thenable = {
  then(resolve) {
    console.log('被拆箱了');
    resolve('内部值');
  },
};

// 原生 Promise.resolve 会「拆箱」thenable
Promise.resolve(thenable).then((v) => console.log(v));
// 被拆箱了
// 内部值
```

所以 `Promise.resolve()` 不是简单的包装，它会**递归拆箱 thenable**。

总结一下，`Promise.resolve()` 有四种情况：

| 参数类型 | 行为 |
|---------|------|
| Promise 实例 | 原封不动返回 |
| thenable 对象（有 `.then` 方法） | 拆箱执行，取最终值 |
| 普通值 / 非 thenable 对象 | 包装成 fulfilled Promise |
| 无参数 | 返回 `Promise.resolve(undefined)` |

```js
// 四种情况对比
const p1 = Promise.resolve(Promise.resolve(1)); // 参数是 Promise → 直接返回，不包两层
p1.then(console.log); // 1

const p2 = Promise.resolve({ then: (r) => r(42) }); // 参数是 thenable → 拆箱
p2.then(console.log); // 42

const p3 = Promise.resolve('hello'); // 参数是普通值 → 直接包装
p3.then(console.log); // hello

const p4 = Promise.resolve(); // 无参数 → undefined 的 resolved
p4.then(console.log); // undefined
```

### 测试 thenable 拆箱

```js
const fakePromise = {
  then: (resolve) => setTimeout(() => resolve('拆出来的值'), 500),
};

MyPromise.resolve(fakePromise).then((v) => console.log(v));
// 500ms 后：拆出来的值 ✅
```

### 2.2 Promise.reject()

```js
MyPromise.reject = function (reason) {
  return new MyPromise((resolve, reject) => reject(reason));
};
```

**关键区别：`reject()` 不拆箱 thenable！**

```js
const thenable = {
  then(resolve) {
    resolve('这个值不会被拆出来');
  },
};

Promise.reject(thenable).catch((e) => {
  console.log(e === thenable); // true — 原样返回，没有拆箱！
});
```

### 对比 resolve 和 reject

```js
const obj = { then: (resolve) => resolve('拆箱') };

// resolve：拆箱
MyPromise.resolve(obj).then((v) => console.log('resolve:', v));
// resolve: 拆箱

// reject：不拆箱
MyPromise.reject(obj).catch((v) => console.log('reject:', v === obj));
// reject: true
```

### 2.3 Promise.all()

```js
MyPromise.all = function (promises) {
  return new MyPromise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    const results = [];
    let settledCount = 0;
    const len = promises.length;

    // 空数组直接 resolve
    if (len === 0) return resolve(results);

    promises.forEach((promise, index) => {
      // 用 MyPromise.resolve 包装，兼容非 Promise 值
      MyPromise.resolve(promise).then(
        (value) => {
          // 结果按原顺序保存，而不是.push
          results[index] = value;
          settledCount++;

          // 全完成了才 resolve
          if (settledCount === len) {
            resolve(results);
          }
        },
        // 任何一个失败，立即 reject
        (reason) => reject(reason)
      );
    });
  });
};
```

三个防坑点：
1. **`results[index]` 而不是 `results.push()`**：保证输出顺序和输入顺序一致
2. **`settledCount` 计数器**：不能用 `results.length`，因为 `results[index]` 可能产生稀疏数组
3. **空数组直接 resolve**：符合规范

### Promise.all 实战：批量请求

```js
function fetchUser(id) {
  return new MyPromise((resolve) => {
    setTimeout(() => resolve({ id, name: `用户${id}` }), 1000 - id * 100);
  });
}

const start = Date.now();
MyPromise.all([
  fetchUser(1),
  fetchUser(2),
  fetchUser(3),
  '直接传字符串也行', // 非 Promise 值
]).then((results) => {
  console.log(`耗时: ${Date.now() - start}ms`);
  console.log(results);
  // 耗时: ~900ms（三个请求并发，取最长那个）
  // [
  //   { id: 1, name: '用户1' },
  //   { id: 2, name: '用户2' },
  //   { id: 3, name: '用户3' },
  //   '直接传字符串也行'
  // ]
});
```

### 注意：个体 catch 会阻止 all 的 catch

一个坑：`Promise.all` 里某个 Promise 如果自己配了 `.catch()`，all 的 `.catch()` 就收不到。

```js
const p1 = Promise.resolve('hello');
const p2 = new Promise((resolve, reject) => {
  throw new Error('p2 出错了');
}).catch((e) => e); // ← p2 自己的 catch 把错误吞了！

Promise.all([p1, p2])
  .then((result) => console.log('all then:', result))
  .catch((e) => console.log('all catch:', e.message));
// 输出：all then: ['hello', Error: p2 出错了]
// p2 的 catch 恢复了状态 → all 认为 p2 成功了
```

**all 只看每个 Promise 的最终状态。单独 catch 了，all 就认为是搞定了。** 想在 all 级别统一处理错误，就不要给每个 Promise 单独加 catch。

### 2.4 Promise.race()

```js
MyPromise.race = function (promises) {
  return new MyPromise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    // 谁先 settled，就直接用谁的结果
    promises.forEach((promise) => {
      MyPromise.resolve(promise).then(resolve, reject);
    });
  });
};
```

race顾名思义就是赛跑，**第一个完成的决定结果**，不管成功还是失败。

### 小demo：实现请求超时

race 最常见的实战场景就是**请求超时控制**：

```js
function withTimeout(promise, ms) {
  const timeout = new MyPromise((resolve, reject) => {
    setTimeout(() => reject(new Error(`超时了！超过 ${ms}ms`)), ms);
  });
  return MyPromise.race([promise, timeout]);
}

// 模拟一个慢请求
const slowRequest = new MyPromise((resolve) => {
  setTimeout(() => resolve('数据来了'), 3000);
});

withTimeout(slowRequest, 1000)
  .then(console.log)
  .catch((e) => console.error(e.message));
// 1 秒后：超时了！超过 1000ms
```

### 2.5 Promise.allSettled()

```js
MyPromise.allSettled = function (promises) {
  return new MyPromise((resolve) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    const results = [];
    let settledCount = 0;
    const len = promises.length;

    if (len === 0) return resolve(results);

    promises.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        (value) => {
          results[index] = { status: 'fulfilled', value };
          settledCount++;
          if (settledCount === len) resolve(results);
        },
        (reason) => {
          results[index] = { status: 'rejected', reason };
          settledCount++;
          if (settledCount === len) resolve(results);
        }
      );
    });
  });
};
```

和 `all` 的区别：**不会因为一个失败就 reject**，等所有都 settled 后才 resolve，返回每个的结果对象。

### 测试 allSettled

```js
MyPromise.allSettled([
  MyPromise.resolve('成功'),
  MyPromise.reject('失败'),
  MyPromise.resolve(42),
]).then((results) => {
  console.log(results);
  // [
  //   { status: 'fulfilled', value: '成功' },
  //   { status: 'rejected', reason: '失败' },
  //   { status: 'fulfilled', value: 42 }
  // ]
});
```

### 2.6 Promise.any()

```js
MyPromise.any = function (promises) {
  return new MyPromise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    const errors = [];
    let rejectedCount = 0;
    const len = promises.length;

    if (len === 0) {
      return reject(new AggregateError(errors, 'All promises were rejected'));
    }

    promises.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        // 有任何一个成功就 resolve
        (value) => resolve(value),
        (reason) => {
          errors[index] = reason;
          rejectedCount++;
          if (rejectedCount === len) {
            reject(new AggregateError(errors, 'All promises were rejected'));
          }
        }
      );
    });
  });
};
```

`any` = **只要有一个成功就成功，全失败才失败**，返回 `AggregateError`。

### 测试 any

```js
// 同时请求三个 CDN，哪个先成功用哪个
const cdn1 = new MyPromise((resolve, reject) => setTimeout(reject, 500, 'CDN1 挂了'));
const cdn2 = new MyPromise((resolve) => setTimeout(resolve, 300, 'CDN2 的数据'));
const cdn3 = new MyPromise((resolve, reject) => setTimeout(reject, 800, 'CDN3 挂了'));

MyPromise.any([cdn1, cdn2, cdn3]).then((result) => {
  console.log('最快的 CDN 返回:', result);
  // 300ms 后：最快的 CDN 返回: CDN2 的数据
});
```

### 五个静态方法一览

| 方法 | 成功条件 | 失败条件 | 返回值 |
|------|---------|---------|--------|
| `all` | 全部成功 | 任意一个失败 | 结果数组 / 失败原因 |
| `race` | 第一个 settled | 第一个 settled | 第一个 settled 的值/原因 |
| `allSettled` | 全部 settled（不会失败） | — | `[{status, value/reason}]` |
| `any` | 任意一个成功 | 全部失败 | 成功的值 / AggregateError |
| `resolve/reject` | — | — | 包装后的 Promise |

---

## 第三章：async/await 的本质 —— Generator + Promise

### 3.1 先看 Generator 是啥

如果你没怎么用过 Generator，先快速过一下：

```js
function* gen() {
  const a = yield 1;
  console.log('a:', a);
  const b = yield 2;
  console.log('b:', b);
  return 3;
}

const g = gen();
console.log(g.next());    // { value: 1, done: false }  ← 执行到第一个 yield，暂停
console.log(g.next(10));  // a: 10 → { value: 2, done: false }  ← 把 10 传回 yield
console.log(g.next(20));  // b: 20 → { value: 3, done: true }
```

Generator 的厉害之处：**函数可以暂停执行，然后从外部把值传回去继续执行。**

### 3.2 async/await 和 Generator 的对应关系

两段代码一起看：

```js
// async/await 写法
async function fetchData() {
  const user = await fetchUser();
  const orders = await fetchOrders(user.id);
  return orders;
}

// Generator 写法
function* fetchDataGen() {
  const user = yield fetchUser();
  const orders = yield fetchOrders(user.id);
  return orders;
}
```

看起来很像哈，区别是：
- `async` ↔ `function*`
- `await` ↔ `yield`

### 3.3 写一个自动执行器：spawn 函数

Generator 的问题是：需要手动调 `.next()` 才能往下走。但如果每次 `yield` 的都是 Promise，就可以写一个**自动执行器**，Promise 完成后自动调 `.next()`：

```js
function spawn(genFn) {
  // spawn 返回一个 Promise
  return new Promise((resolve, reject) => {
    const gen = genFn();

    function step(nextFn) {
      let next;
      try {
        next = nextFn();
      } catch (e) {
        // Generator 内部抛异常，直接 reject
        return reject(e);
      }

      // Generator 执行完了
      if (next.done) {
        return resolve(next.value);
      }

      // 还没完：把 yield 的值包装成 Promise
      // 等待它完成后，把结果传回 Generator 继续执行
      Promise.resolve(next.value).then(
        (value) => step(() => gen.next(value)),
        (reason) => step(() => gen.throw(reason))
      );
    }

    // 启动！
    step(() => gen.next());
  });
}
```

### 3.4 用 spawn 运行 Generator

```js
function fetchUser() {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ id: 1, name: '大熊猫' }), 500);
  });
}

function fetchOrders(userId) {
  return new Promise((resolve) => {
    setTimeout(() => resolve(['订单A', '订单B', '订单C']), 500);
  });
}

spawn(function* () {
  console.log('开始请求...');
  const user = yield fetchUser();
  console.log('用户:', user.name);
  const orders = yield fetchOrders(user.id);
  console.log('订单:', orders);
  return orders;
}).then((result) => {
  console.log('最终结果:', result);
});

// 开始请求...
// (500ms) 用户: 大熊猫
// (再 500ms) 订单: ['订单A', '订单B', '订单C']
// 最终结果: ['订单A', '订单B', '订单C']
```

这就是 async/await 的核心秘密：**`async function` 本质上就是 `spawn(function* () { ... })`**。

### 3.5 spawn 的三个关键步骤

```
1. 调 gen.next() 让 Generator 开始执行
          ↓
2. 遇到 yield → 暂停  →  拿到 yield 后面的 Promise
          ↓
3. Promise.resolve(yieldValue).then(value => gen.next(value))
          ↓
   回到步骤 2，直到 done === true
```

**async/await 只是这个过程的语法糖**。V8 引擎里的实际实现比这个复杂（有性能优化），但核心思路是一致的。

### 尝试给 spawn 加 try/catch 支持

```js
spawn(function* () {
  try {
    const user = yield fetchUser();
    const orders = yield Promise.reject(new Error('订单接口崩了'));
    console.log(orders); // 不会执行
  } catch (e) {
    console.error('捕获到了:', e.message);
    // 捕获到了: 订单接口崩了
  }
});
```

代码能正常工作，因为 `spawn` 里用 `gen.throw(reason)` 处理了 Promise 失败的情况，Generator 内部的 try/catch 可以捕获到。

### 3.6 补充：Promise.try() —— 统一同步/异步错误处理

实际开发中经常遇到一个问题：一个函数可能是同步也可能是异步，但我们想用统一的 `.then/.catch` 处理。

```js
// 问题：f 可能是同步函数，也可能返回 Promise
function process(f) {
  // 写法一：Promise.resolve().then(f)
  // 问题：如果 f 是同步函数，它会推迟到微任务才执行
  Promise.resolve().then(f);
  console.log('next');
  // 输出：next → f() 的结果（被推迟了！）

  // 写法二：用 async
  (async () => f())();
  // 问题：如果 f 抛同步错，(async () => f())() 会返回 rejected Promise，
  // 外面 try-catch 抓不到
}
```

ES2025 引入了 `Promise.try()` 解决这个问题：

```js
const f = () => {
  console.log('now');
  return 'done';
};

Promise.try(f).then(console.log);
console.log('next');
// now
// next
// done
// ↑ 同步函数同步执行，不推迟！
```

对于返回 Promise 的异步函数，也能正确处理：

```js
const asyncFn = () => {
  return new Promise((resolve) => {
    setTimeout(() => resolve('async result'), 1000);
  });
};

Promise.try(asyncFn).then(console.log);
// 1 秒后：async result
```

最关键的：**同步错误和异步错误，全进 `.catch()`**。

```js
// 以前需要 try-catch + .catch 两套处理
function dangerousFn() {
  // 可能同步抛错
  if (Math.random() > 0.5) throw new Error('同步错误');
  // 可能返回 rejected Promise
  return fetch('/api').then((r) => r.json());
}

// 错误：必须这样写——又 try-catch 又 .catch，啰嗦
// try {
//   dangerousFn().then(...).catch(...);
// } catch(e) { ... }

// 正确：Promise.try 一把搞定
Promise.try(dangerousFn)
  .then(handleResult)
  .catch(handleError); // 同步错误、异步错误，全进这里！
```

**`Promise.try()` 就是模拟 try 代码块，`promise.catch()` 就是模拟 catch 代码块。让所有操作都有统一的错误处理机制。**

> 事实上，Bluebird、Q、when 这些 Promise 库早就提供了 `.try()` 方法，现在终于进了 ES2025 标准。

---

## 第四章：Promise 并发控制

面试官特别喜欢问：**「请实现一个函数，控制 Promise 的并发数量」**。

场景：有 100 个 URL 要请求，但浏览器同域名最多只能有 6 个并发连接。

```js
/**
 * 并发控制函数
 * @param {Array} tasks - 返回 Promise 的任务函数数组
 * @param {number} limit - 最大并发数
 * @returns {Promise<Array>} - 按原顺序返回结果
 */
function asyncPool(tasks, limit) {
  const results = [];
  let running = 0;     // 当前正在执行的请求数
  let nextIndex = 0;   // 下一个要执行的任务下标

  return new Promise((resolve) => {
    function runNext() {
      // 全部完成（所有任务都执行完 + 没有在跑的了）
      if (nextIndex >= tasks.length && running === 0) {
        return resolve(results);
      }

      // 在并发限制内，且还有未执行的任务时，持续启动
      while (running < limit && nextIndex < tasks.length) {
        const curIndex = nextIndex;
        nextIndex++;
        running++;

        tasks[curIndex]()
          .then((data) => {
            results[curIndex] = data;
          })
          .catch((err) => {
            results[curIndex] = err;
          })
          .finally(() => {
            running--;
            runNext(); // 一个任务结束，尝试启动下一个
          });
      }
    }

    runNext();
  });
}
```

### 完整测试：模拟并发请求

```js
// 模拟 20 个耗时不同的「请求」
const tasks = Array.from({ length: 20 }, (_, i) => {
  return () =>
    new Promise((resolve) => {
      const delay = 500 + Math.random() * 1500;
      setTimeout(() => resolve(`任务${i + 1} (耗时${delay.toFixed(0)}ms)`), delay);
    });
});

// 限制最多 3 个并发
const start = Date.now();
asyncPool(tasks, 3).then((results) => {
  console.log(`总耗时: ${((Date.now() - start) / 1000).toFixed(1)}s`);
  console.log('结果:', results);
  // 总耗时约 4-5s（比串行 20+ 秒快多了）
});
```

### 画出来

```
时间线（limit = 3）：

0ms:    [任务1] [任务2] [任务3]   ← 三个同时启动
300ms:  [任务4] [任务2] [任务3]   ← 任务1完成，任务4启动
500ms:  [任务4] [任务5] [任务3]   ← 任务2完成，任务5启动
...     始终保持 3 个在执行
```

---

### 4.1 进阶：加上重试 + 指数退避

并发控制解决了「别把服务打爆」的问题。但如果某个请求因为网络抖动失败了，直接抛错就有点浪费——给它一次重试机会可能就活了。

把重试逻辑封装到并发控制里：

```js
/**
 * 判断错误是否可以重试（网络超时、服务不可用可以重试；参数错误、404 不重试）
 */
function isRetryableError(err) {
  const retryableCodes = ['ETIMEDOUT', 'ECONNRESET', 'ECONNREFUSED', '503', '429'];
  return retryableCodes.some((code) => err.message?.includes(code) || err.code === code);
}

/**
 * 并发控制 + 自动重试
 * @param {Array} tasks - 返回 Promise 的任务函数数组
 * @param {number} limit - 最大并发数
 * @param {number} maxRetries - 每个任务最大重试次数
 * @returns {Promise<Array>}
 */
function asyncPoolWithRetry(tasks, limit, maxRetries = 2) {
  const results = [];
  let running = 0;
  let nextIndex = 0;

  async function executeTask(index, retriesLeft) {
    try {
      results[index] = await tasks[index]();
    } catch (err) {
      if (retriesLeft > 0 && isRetryableError(err)) {
        console.warn(`任务 ${index + 1} 失败，剩余重试 ${retriesLeft} 次`);
        // 指数退避：第 1 次重试等 1s，第 2 次等 2s
        await new Promise((r) => setTimeout(r, 1000 * (maxRetries - retriesLeft + 1)));
        return executeTask(index, retriesLeft - 1);
      }
      results[index] = err;
    }
  }

  return new Promise((resolve) => {
    function runNext() {
      if (nextIndex >= tasks.length && running === 0) return resolve(results);
      while (running < limit && nextIndex < tasks.length) {
        const curIndex = nextIndex++;
        running++;
        executeTask(curIndex, maxRetries).finally(() => {
          running--;
          runNext();
        });
      }
    }
    runNext();
  });
}
```

几个关键设计：
1. **区分可重试/不可重试错误**：网络超时值得重试，参数校验错误重试也白搭
2. **指数退避**：不要马上重试，等 1s → 2s → 4s，避免雪崩
3. **重试上限**：不是无限重试，最多 2~3 次

---

## 第五章：手写 Promise 的实战价值

你可能会想：**「我又不是去给 V8 写代码，手写 Promise 有什么用？」**

几个实打实的受益场景：

### 5.1 面试

当面试官问「Promise 的原理是什么」，很多人都会背出「三种状态、链式调用」诸如此类。但是如果能从构造函数写到 resolvePromise 递归，从微任务写到 async/await 实现——**这个回答质量差了三个档次**。

### 5.2 读懂开源项目源码

很多库内部实现了自己的 Promise 或类似机制。比如：
- **axios** 的拦截器就是基于 Promise 链
- **Koa** 的洋葱模型本质是 Promise 组合
- **Redux-saga** 的核心就是 Generator + Promise
- 各种 **数据库 ORM** 的查询链，本质就是 then 链

手写过后，再看这些库，你看的是「原来这就是 resolvePromise 的变体」，而不是心里想「这什么玩意啊」。

### 5.3 写出更好的异步代码

理解微任务后，就会知道为什么这样写不对：

```js
// 错误：以为会按顺序，其实 Promise.all 是并发的
for (const url of urls) {
  const data = await fetch(url); // 串行的！
}
// 正确：用 all 或 asyncPool 并发
```

理解 then 返回新 Promise 后，就会知道拦截器为什么这样设计：

```js
// axios 拦截器的本质就是 Promise 链
axios.interceptors.request.use(
  (config) => { /* 改 config */ return config; },
  (error) => Promise.reject(error)
);
```

### 5.4 复杂业务流程编排 — Workflow/Saga 模式

实际业务不只是「取数据→展示」这么简单。场景是：**创建订单前必须校验库存，库存锁定后扣减余额，余额扣完创建订单，任何一步失败都要回滚前面的操作。**

这是典型的「事务流」：顺序执行多个异步步骤，外加补偿/回滚。直接写 try-catch 会又臭又长：

```js
// ❌ 直接写：每一步 try-catch + 回滚，代码会爆炸
async function createOrder(ctx) {
  try {
    await checkInventory(ctx);
  } catch (e) {
    // 没有前置操作，无需回滚
    throw e;
  }

  try {
    await deductBalance(ctx);
  } catch (e) {
    // 回滚：释放库存
    await releaseInventory(ctx);
    throw e;
  }

  try {
    await saveOrder(ctx);
  } catch (e) {
    // 回滚：退款 + 释放库存
    await refundBalance(ctx);
    await releaseInventory(ctx);
    throw e;
  }
}
```

步骤越多，回滚代码越膨胀——而且回滚逻辑跟业务逻辑耦合在一起，改一步得改好几天。

**用 Workflow 编排模式解耦**：每个步骤自带补偿函数，引擎负责按序执行 + 失败自动回滚。

```js
class WorkflowEngine {
  constructor() {
    this.steps = [];
  }

  // addStep(name, 正向操作, 补偿/回滚操作)
  addStep(name, executor, compensator) {
    this.steps.push({ name, executor, compensator });
    return this; // 支持链式配置
  }

  async execute(context) {
    const completedSteps = []; // 记录已完成的步骤，回滚时用

    for (const step of this.steps) {
      try {
        console.log(`执行：${step.name}`);
        await step.executor(context);
        completedSteps.push(step);
      } catch (error) {
        console.error(`${step.name} 失败，开始回滚...`);
        // 逆序执行补偿（后完成的先回滚）
        await this._rollback(completedSteps, context);
        throw error; // 回滚完抛出原始错误
      }
    }
    return context;
  }

  async _rollback(completedSteps, context) {
    for (const step of completedSteps.reverse()) {
      if (!step.compensator) continue; // 无补偿函数的步骤跳过
      try {
        await step.compensator(context);
      } catch (e) {
        console.error(`回滚 ${step.name} 失败:`, e.message);
        // 回滚失败不中断，继续回滚后续步骤
      }
    }
  }
}

// 使用：订单创建流程
const orderWorkflow = new WorkflowEngine()
  .addStep('CheckStock', checkInventory, releaseInventory)
  .addStep('DeductBalance', deductMoney, refundMoney)
  .addStep('CreateOrder', saveOrderData, cancelOrder)
  .addStep('SendNotification', sendMsg, null); // 通知失败不回滚

await orderWorkflow.execute({ orderId: '123', amount: 100 });
```

这个模式的核心思想：**业务逻辑与流程控制解耦**。新增步骤只需 `addStep`，不用在 try-catch 里改回滚逻辑。`axios` 拦截器的链式、`Koa` 的洋葱模型、`Redux-saga` 的 effect 编排——都是同一套思想的不同变体。

### 5.5 异步测试的稳定写法

异步代码的测试特别容易「假绿」——测试通过了不是因为代码对，是因为断言跑在了异步代码之前。

常见的错误写法：

```js
// ❌ 用固定 sleep 等异步结果——脆且慢
test('should update status', async () => {
  triggerAsyncProcess();
  await sleep(2000); // 2 秒够吗？机器负载高时不够；闲置时浪费
  expect(await getStatus()).toBe('completed');
});
```

**正确做法：轮询断言，直到条件满足或超时。**

```js
/**
 * 智能等待函数：轮询检查条件，直到满足或超时
 * @param {Function} condition - 返回 boolean 或 truthy 值的检查函数
 * @param {number} timeout - 最大等待时间 ms
 * @param {number} interval - 检查间隔 ms
 */
async function waitFor(condition, timeout = 5000, interval = 100) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    if (await condition()) return;
    await new Promise((r) => setTimeout(r, interval));
  }
  throw new Error(`Timeout after ${timeout}ms`);
}

// ✅ 正确：等待状态真正变为 'completed'
test('should update status', async () => {
  triggerAsyncProcess();

  await waitFor(async () => {
    const status = await getStatus();
    return status === 'completed';
  });

  expect(await getFinalResult()).toBe('success');
});
```

几个要点：
1. **不要硬编码 sleep**：不同机器、不同负载下执行时间不同，固定等待要么太短要么浪费
2. **检查间隔不宜太密**：100ms 够用，不频繁消耗 CPU
3. **超时报错**：别让测试无限等，5 秒超时够用了
4. **用 Jest/Vitest 的话可以用 `waitFor`**（已内置），不用自己封装

---

## 总结

```
第一篇（mini Promise）         第二篇（完整规范）           第三篇（内核剖析）
                                                          
 状态机 + then                then 返回新 Promise        微任务 vs 事件循环
      ↓                            ↓                        ↓
 回调队列（支持异步）          值穿透 + 异常处理           5 个静态方法
      ↓                            ↓                        ↓
 catch + finally              resolvePromise 解析         async/await = Generator
                                   ↓                   + Promise 自动执行器
                              872 测试全绿                   ↓
                                                        并发控制 asyncPool
```

1. **Promise 是状态机**，状态不可逆，保证了一次性
2. **then 返回新 Promise** 是链式调用的根基，每个 then 都是新实例
3. **resolvePromise** 处理 thenable 递归，是规范中最精妙的部分
4. **微任务队列** 在每个宏任务后清空，Promise 回调借此「慢一步」但优先于下一个宏任务
5. **async/await = Generator + 自动执行器**，不是魔法
6. **手写不是为了造轮子**，是为了真正理解，面试、读源码、写代码都会受益

---

## Promise 面试解答

```
Q: Promise 的状态？
A: pending/fulfilled/rejected，只可变一次

Q: then 为什么能链式调用？
A: then 返回新 Promise（promise2），回调返回值决定 promise2 的状态

Q: 什么是 resolvePromise？
A: 处理回调返回值 x 是 thenable 的情况，递归解析直到拿到普通值

Q: Promise 回调和 setTimeout 谁先执行？
A: Promise 回调是微任务，在当前宏任务结束后立即执行；setTimeout 是宏任务，在微任务之后

Q: async/await 怎么实现的？
A: async = function* + spawn 自动执行器; await = yield; spawn 用 Promise.resolve + .then 递归驱动

Q: Promise.all vs allSettled vs any？
A: all = 全成功才成功; allSettled = 全 settled（不会失败）; any = 一个成功就行
```

---

## 相关面试题

### 第 1 题：微任务 vs 宏任务——经典执行顺序

```js
console.log('start');

setTimeout(() => console.log('timeout'), 0);

Promise.resolve().then(() => {
  console.log('then1');
}).then(() => {
  console.log('then2');
});

console.log('end');
```

输出顺序？


```
start
end
then1
then2
timeout
```

执行流程：
1. 同步代码：`start`、`end`
2. 微任务队列清空：`then1`、`then2`（then1 产生的 then2 也在本轮微任务清空）
3. 下一轮事件循环，宏任务执行：`timeout`

**每个宏任务执行完后，必须清空微任务队列，才轮到下一个宏任务。**

---

### 第 2 题：嵌套 Promise 的执行顺序

```js
new Promise((resolve) => {
  console.log(1);
  resolve();
}).then(() => {
  console.log(2);
  new Promise((resolve) => {
    console.log(3);
    resolve();
  }).then(() => {
    console.log(4);
  });
}).then(() => {
  console.log(5);
});
```

输出顺序？


```
1
2
3
4
5
```

分析：
1. 同步：打印 1
2. 微任务 then1：打印 2，同步执行 new Promise 打印 3，then4 入微任务队列
3. then1 执行完，then5 入微任务队列
4. 微任务队列现在是 `[then4, then5]`，按顺序执行：4 → 5

关键点：then1 里的 new Promise 是同步的，产生的新 .then(4) 排到微任务队列末尾。

---

### 第 3 题：Promise.resolve 的 thenable 陷阱

```js
Promise.resolve({
  then: (resolve, reject) => {
    resolve('thenable!');
  },
}).then(console.log);
```

输出什么？


输出：`thenable!`

`Promise.resolve` 看到参数有 `.then` 方法时，会调用它并跟随其最终状态。这就是 thenable 拆箱。

反过来：
```js
Promise.resolve(42).then(console.log); // → 42
```
参数是原始值，直接 resolve 为 42。

---

### 第 4 题：all / race / allSettled / any 四兄弟的区别

简要说明四个静态方法的行为差异。


| 方法 | 成功条件 | 失败条件 | 返回值 |
|------|---------|---------|--------|
| `all` | 全部成功 | 一个失败 | 结果数组 / 第一个错误 |
| `race` | 第一个 settled | 第一个 settled | 第一个 settled 的值 |
| `allSettled` | 全部 settled | **永不 reject** | `[{status, value/reason}]` |
| `any` | 任意一个成功 | 全部失败 | 第一个成功值 / AggregateError |

面试最常考 `all` vs `allSettled`：all 一个失败全崩，allSettled 全跑完再汇报。

---

### 第 5 题：async/await 的本质是什么？

用一句话解释 `async/await` 是什么的语法糖。


**async/await 是 Generator + Promise 自动执行器的语法糖。**

```js
// async function
async function foo() {
  const a = await Promise.resolve(1);
  const b = await Promise.resolve(2);
  return a + b;
}

// 本质上等价于
function foo() {
  return spawn(function* () {
    const a = yield Promise.resolve(1);
    const b = yield Promise.resolve(2);
    return a + b;
  });
}
```

`async` 把函数返回值包装成 Promise，`await` 相当于 `yield`，`spawn` 自动在每次 yield 完成后调 `.next()`。

---

### 第 6 题：手写题——实现 Promise.all

请手写一个 `myAll` 函数，输入 Promise 数组，返回一个 Promise。


```js
function myAll(promises) {
  return new Promise((resolve, reject) => {
    if (!promises.length) return resolve([]);

    const results = [];
    let settledCount = 0;

    for (let i = 0; i < promises.length; i++) {
      Promise.resolve(promises[i]).then(
        (value) => {
          results[i] = value;
          settledCount++;
          if (settledCount === promises.length) resolve(results);
        },
        (reason) => reject(reason)
      );
    }
  });
}
```

容易错误的点：
1. 用 `results[i]` 而不是 `results.push()`——保证顺序
2. 用 `settledCount` 判全完成，不能用 `results.length`（稀疏数组的坑）

欢迎关注微信公众号：程序员蜡笔熊，期待与您的讨论！

![image-weixin](/img/weixinpublic.jpg)