---
pubDatetime: 2024-02-21T12:26:18.269Z
title: JavaScript 编码题 part1
featured: true
draft: false
tags:
  - 前端
  - 编码题
  - JavaScript
description: 本篇编码实现包含深拷贝、防抖和节流、Promise.all & race 以及函数的 apply/call/bind。
---

coding 在当下前端面试中属于是必考项目。最近在学习前端知识，准备面试，在各个渠道遇到了一些有代表的 code 题目，整理记录如下。本篇包含深拷贝、防抖和节流、Promise.all & race 以及函数的 apply/call/bind。

## Table of contents

## 1. 浅拷贝与深拷贝

浅拷贝一般不会直接考，为了防止失忆，这里还是要搂一眼

```javascript
let obj1 = { a: 1, b: { c: 3 } };
// 1. 手动赋值
let obj2 = { a: obj1.a, b: obj1.b };
// 2. 展开运算符
let obj3 = { ...obj1 };
// 3. assign
let obj4 = Object.assign({}, obj1);
```

深拷贝主要是解决了：浅拷贝对象中属性为引用类型时引用值复用的问题。

最简单且直观的方式就是通过 JSON

```javascript
// 1. JSON.stringfy parse，只能用于可序列化的对象，function、HTML 元素不行
let obj5 = JSON.parse(JSON.stringify(obj1));
```

还有就是直接使用浏览器 API：structuredClone

```javascript
// 2. 浏览器环境 API 方法 structuredClone，也只针对可序列化对象
let obj6 = structuredClone(obj1);
```

当然如注释所言，以上两种方法对于简单对象是完全够用了，但是都是只应用于对象是是可序列化，如果有 function 或 HTMLElement 时无法完成拷贝，表现为 JSON.stringfy 会跳过该属性，structuredClone 会报异常。

下面是一个考虑了多种情况的递归式深拷贝的实现，同时为了解决循环引用的问题，引入了一个 map 用来缓存已经拷贝过的对象。

```javascript
function deepClone(obj, map = new Map()) {
  // 首先判断 obj 是否为对象或数组（基本类型直接返回）
  if (typeof obj !== "object" || obj === null) {
    return obj;
  }

  // 如果已经拷贝过该对象，则直接返回拷贝后的对象
  if (map.has(obj)) {
    return map.get(obj);
  }

  // 创建一个新的对象或数组来存储拷贝的值
  //   const clone = Array.isArray(obj) ? [] : {}
  // 创建一个新的对象或数组来存储拷贝的值，并将原对象的原型设置为新对象的原型
  const clone = Object.create(Object.getPrototypeOf(obj));

  // 透过 Ojbect.entries 来迭代，然后递回地对每个值深拷贝
  // 因为 Object.entries 不会列举整个原型链 (prototype chain)
  // 所以不用透过 obj.hasOwnProperty(key) 额外检查是不是非原型链上的属性
  for (const [key, value] of Object.entries(obj)) {
    clone[key] = deepClone(value, map);
  }

  // 将当前对象存储到 map 中
  map.set(obj, clone);

  return clone;
}
```

## 2. 手写 Promise.all 和 race 方法

这两个方法入参是一个 promises 数组，返回的都是一个新的 Promise 对象，区别是：

- all 在所有给定的 promise 都完成（fullfilled）时，返回的 promise 对象才会 fullfilled，否则只要任一个入参 promise 被拒绝（rejected），该结果也会 rejected。
- **race** 返回的 promise 对象在给定的任意一个入参 Promise 变为 fulfilled 或 rejected 时变为相应状态。

```javascript
Promise.myAll = function (promiseArr) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completeCount = 0;

    promiseArr.forEach((promise, index) => {
      promise.then(
        vaule => {
          results[index] = vaule; // 结果保持 promise 顺序
          completeCount++;

          if (completeCount === promiseArr.length) {
            resolve(results);
          }
        },
        reason => {
          reject(reason);
        }
      );
    });
  });
};

Promise.myRace = function (promiseArr) {
  return new Promise((resolve, reject) => {
    promiseArr.forEach(promise => {
      promise.then(
        value => {
          resolve(value); // 任一个先 fullfill 直接 resolve 最外部的 promise
        },
        reason => {
          reject(reason);
        }
      );
    });
  });
};
```

## 3. 防抖和节流

防抖函数的目标是确保在某个连续动作结束后的一段时间内只执行一次函数。例如，防止用户在输入框中频繁输入触发搜索。

防抖的关键是【延迟执行，最后一次】

防抖的原理是在一定时间内，只有最后一次触发事件后才执行函数。

```javascript
function debounce(func, delay) {
  let timeoutId;

  return function (...args) {
    if (timeoutId) {
      // 如果 timeoutId 存在（之前的定时器仍在运行），则清除之前的定时器
      clearTimeout(timeoutId);
      timeoutId = null; // 这里需要手动重置 ID，因为 clearTimeout 并不会处理这个变量引用，参见 MDN
    }
    // 设置一个新的定时器，在指定的延迟后执行函数
    timeoutId = setTimeout(() => {
      func.apply(this, args); // 确保能正确传递 func 被调用时的上下文（如果 func 内部使用了 this 的话就会体现）
    }, delay);
  };
}

// --- tests ---
const debounceBtn = document.getElementById("debounce");
const debounceFunc = debounce(function () {
  console.log("debounce clicked");
}, 1000);
debounceBtn.addEventListener("click", () => {
  debounceFunc();
});
```

节流函数的目标是确保在一段时间内只执行一次函数。例如，防止用户在滚动页面时触发过多的事件。

节流的关键是【立即执行，第一次】

节流的原理是在一定时间内，固定的时间间隔只执行一次函数。

```javascript
function throttle(func, delay) {
  let isThrottled = false;
  return function (...args) {
    // 如果 isThrottled 为 false，执行函数并将 isThrottled 设置为 true
    if (!isThrottled) {
      func.apply(this, args);
      isThrottled = true;

      // 同时设置一个延迟后将 isThrottled 设置回 false，使得在下一个时间间隔内可以再次触发函数
      setTimeout(() => {
        isThrottled = false;
      }, delay);
    }
  };
}

// --- tests ---
const throttleBtn = document.getElementById("throttle");
const throttleFunc = throttle(() => {
  console.log("throttle clicked");
}, 500);
throttleBtn.addEventListener("click", () => {
  throttleFunc();
});
```

补充：`func.apply(this, args)` 中的 `this` 将指向调用防抖或节流函数时的上下文，而 `args` 将是调用防抖或节流函数时传递的参数。这样可以确保在执行目标函数时，保持原有的上下文和参数。

## 4. 实现 apply/call/bind

JavaScript 中的 `apply`、`call` 和 `bind` 是用于改变函数执行上下文（`this` 指向）的方法。

区别在于：

- apply：立即调用函数，传递一个数组，里面是任意数量的参数；
- call：立即调用函数，传递任意数量的参数（以逗号分割）；

- bind：绑定一个新函数，但不立即执行，预先指定 this 值，传入任意数量的参数（以逗号分割）；返回值为一个函数。

apply 和 call 逻辑基本一致，除了入参的形式不同，这里为了便于理解，我们先写一个原理框架：

```javascript
Function.prototype.myApply = function (thisArg, argsArray) {
  const context = thisArg || window; // 如果未提供 context，则默认为全局对象（浏览器中为 window）
  const uniqueKey = Symbol(); // 创建一个唯一的键，以防止覆盖已有属性
  context[uniqueKey] = this; // 将原始函数绑定到指定的上下文对象上，注意这里 this 就是原始函数，因为调用时 funcName.myApply(...)
  const result = context[uniqueKey](...argsArray); // 调用原始函数，传递指定的参数
  delete context[uniqueKey]; // 调用后删除添加到上下文对象的属性
  return result; // 返回执行结果
};

Function.prototype.myBind = function (thisArg, ...argsArray) {
  const originalFunction = this; // 保存原始函数的引用

  return function (...newArgs) {
    // 注意这里 bind 的不同之处： bind 不是立即执行，所以这里返回一个新函数
    // 而且入参是逗号分割的任意多个，在函数真正执行时还可以继续传入参数，所以这里要重点关注参数的合并
    const mergedArgs = argsArray.concat(newArgs);
    return originalFunction.apply(thisArg, mergedArgs); // 调用原始函数，返回执行结果
  };
};

// --- tests ---
function add(a, b) {
  console.log("add", this);
  return a + b;
}

const obj = { a: 1, b: 2 };
console.log(add.myApply(obj, [3, 4]));

const bindFunc = add.myBind(obj, 3);
console.log("bind", bindFunc);
console.log("bind", bindFunc(4));
```

这些手动实现的方法为了简化，省略了对一些边缘情况的处理：

1. 三个方法的第一步，都要先判断 this 是否是 function，否则抛出异常，终端执行；

   ```javascript
   if (typeof this !== "function") {
     throw new TypeError("myApply was called on which is not a function");
   }
   ```

2. 第一个上下文入参 thisArg，要确保是对象类型；

   ```javascript
   // 2. thisArg 是否有值？无值绑定到 window，有值就转化为对象，然后作为 this
   if (thisArg === undefined || thisArg === null) {
     thisArg = window;
   } else {
     // 确保 thisArg 是一个对象（后续不再附加解释）
     // 如果 thisArg 已经是对象，则操作不会改变它；如果是原始值，它将被转换为相应类型的包装对象。
     thisArg = Object(thisArg);
   }
   ```

3. args 入参是否有值，是否为数组，要分别处理；

   ```javascript
   if (args && args.length) {
     result = thisArg[func](...args);
   } else {
     result = thisArg[func]();
   }
   ```
