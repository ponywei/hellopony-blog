---
pubDatetime: 2024-02-22T15:39:47.825Z
title: JavaScript 编码题 part2
featured: true
draft: false
tags:
  - 前端
  - JavaScript
  - 编码题
  - 面试
description: 本篇编码实现包含 Array.flatten、函数柯里化、new 关键字、instanceOf 关键字，以及实现 LRU Cache。
---

coding 在当下前端面试中属于是必考项目。最近在学习前端知识，准备面试，在各个渠道遇到了一些有代表的 code 题目，整理记录如下。
本篇包含 Array.flatten、函数柯里化、new 关键字、instanceOf 关键字，以及实现 LRU Cache。

## Table of contents

## 1. 实现 Array.flatten

实现对数组的展开 flatten 方法，效果如下

```javascript
flatten([1, [2, 3]]); // [1, 2, 3]
flatten([
  [1, 2],
  [3, 4],
]); // [1, 2, 3, 4]
```

这道题的原型其实就是 [Array.prototype.flat](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/flat)，当然他更复杂一点，因为他还支持传入要展开的层级。

```javascript
// 1. 最简单的方式，mdn 提供，Array 内置方法
// https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/flat
function flatten1(arr) {
  return arr.flat(Infinity);
}

// 2. 如果数组中仅包含数字，那可以借助 Array.prototype.toString 方法
// 因为 toString 如果遇到数组，会继续 toString
function flattenOnlyNumbers(array) {
  return array
    .toString()
    .split(",")
    .map(numStr => Number(numStr));
}

// 3. 循环 + 头插， 不修改原数组
// 借助 Array.isArray 或者 instanceof Array 可以判断元素是否数组
function flattenIterative(array) {
  const result = [];
  const arrayCopy = array.slice();

  while (arrayCopy.length) {
    const item = arrayCopy.shift(); // pop 第一个元素
    if (Array.isArray(item)) {
      arrayCopy.unshift(...item); // 遇到数组，展开一次，然后头插入原数组，下次循环继续处理
    } else {
      result.push(item);
    }
  }

  return result;
}

// 4. 循环头插的变体，借助 some，修改了原数组
function flattenIterativeSome(array) {
  while (array.some(Array.isArray)) {
    array = [].concat(...array); // 如果数组内还存在数组，每次循环展开一个，直至无数组
  }
}

// 5. 递归，借助 reduce
function flattenRecursive(array) {
  return array.reduce((acc, curr) =>
    acc.concat(Array.isArray(curr) ? flattenRecursive(curr) : curr)
  );
}
```

## 2. 函数柯里化

函数柯里化 Currying 是一种将多参数函数转换为一系列单参数函数的技术，例如：

```javascript
// 柯里化前的函数
function add(x, y, z) {
  return x + y + z;
}

// 柯里化后的函数
function curriedAdd(x) {
  return function (y) {
    return function (z) {
      return x + y + z;
    };
  };
}

// 使用柯里化前的函数
const result1 = add(1, 2, 3);
console.log(result1); // 输出：6

// 使用柯里化后的函数
const curriedResult = curriedAdd(1)(2)(3);
console.log(curriedResult); // 输出：6
```

柯里化在函数式编程中很常见，它有以下优点：

- 提前传入部分参数，得到一个待接收剩余参数的新函数，函数的功能更具体；
- 延迟执行，返回的一系列的函数，可以灵活选择执行时机；
- 参数复用，部分应用函数可以多次调用；
- 提高函数组合性。

实现柯里化函数的核心思想就是：每接收一个参数，就返回一个新函数；在这个函数内部，再对剩余参数参数做同样的处理，直至接收到的参数个数等于目标函数参数个数，全部处理完，执行函数。

所以要借助递归来实现，递归结束的关键点是：判断当前参数的数量是否达到了目标函数的参数数量，如果是，则直接调用目标函数，否则返回一个新函数，继续等待接收参数。

```javascript
function curry(func) {
  return function curried(...args) {
    if (args.length >= func.length) {
      // 当前参数的数量达到了目标函数的参数数量，直接执行
      return func.apply(this, args);
    } else {
      // 否则返回一个新的函数，接收剩余的参数，继续递归
      return function (...args2) {
        return curried.apply(this, args.concat(args2));
      };
    }
  };
}

// --- tests ---
function add(a, b, c) {
  return a + b + c;
}

const curriedAdd = curry(add);
const curriedAddOne = curriedAdd(1);
console.log(curriedAdd(1, 2, 3));
console.log(curriedAddOne(2, 3));
console.log(curriedAddOne(2)(3));
```

### PS：函数实例的 length

上面的代码实现里，还有个知识点，就是函数的 length 属性，参见 [MDN：Function实例的 length](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/length)，关键点如下：

- 指 Function 对象形参的个数；

- 不包括剩余参数，就是 ...args 这种写法接收的参数；

- 只包括在第一个具有默认值的参数之前的参数，就是只算有默认值的参数之前的参数个数；

  ```javascript
  function func1(a, b) {...}  // 2
  function func2(a, b = '2') {...}  // 1
  function func3(a = '3', b) {...}  // 0
  function func4(a, ...args) {...}  // 1
  ```

## 3. 实现 new 关键字

`new` 关键字用于创建对象实例，实际上它做了一系列的步骤，包括创建一个新对象、将构造函数的原型连接到新对象、执行构造函数，并返回新对象。

```javascript
function myNew(constructor, ...args) {
  // 创建一个新对象，并将该对象的原型指向构造函数的原型
  const obj = Object.create(constructor.prototype);
  // 将构造函数的 this 绑定到新对象上，并执行构造函数
  const result = constructor.apply(obj, args);
  // 如果构造函数有返回值且是对象类型，则返回该对象；否则返回新创建的对象
  return typeof result === "object" && result !== null ? result : obj;
}

// --- tests ---
function Person(name) {
  this.name = name;
}

const person = myNew(Person, "John");
console.log(person.name, person); // John
```

在第三步判断构造函数执行结果的类型时，要注意判断`result !== null`，因为 `typeof null === 'object'`。

## 4. 实现 instanceOf 关键字

`instanceof` 关键字用于检查对象是否是某个构造函数的实例。它的实现原理是通过检查对象的原型链是否包含构造函数的原型。

```javascript
function myInstanceOf(obj, constructor) {
  // 首先检查参数合法性（instanceof 只能判断对象类型）
  if (obj === null || typeof obj !== "object") {
    return false;
  }

  // 获取对象的原型
  const proto = Object.getPrototypeOf(obj);
  // 遍历原型链，沿着原型链向上查找是否存在构造函数的原型，直至 终点 null
  while (proto) {
    if (proto === constructor.prototype) {
      return true;
    }
    proto = Object.getPrototypeOf(proto); // 继续上溯
  }
  // 没有找到
  return false;
}

// --- tests ---
function Person(name, age) {
  this.name = name;
  this.age = age;
}

const person = new Person("John", 25);
console.log(myInstanceOf(person, Person)); // 输出：true
console.log(myInstanceOf(person, Array)); // 输出：false
```

## 5. 实现 LRU Cache

LRU（Least Recently Used 最近最少使用）缓存是一种常见的缓存策略，它保留最近使用过的数据，而淘汰最久未被使用的数据。

要求：构造函数传入容量 capacity，实现 get 和 put 方法，注意 put 时要控制缓存的容量。

其实这题 [LeetCode](https://leetcode.cn/problems/lru-cache/description/) 上也有，详细的说明可以看链接。

实现的关键是，我们选择一个合适的数据结构，来存储缓存的同时，根据 get 使用操作来维护其使用的新鲜度，在 put 容量满时决定要淘汰的元素。

JavaScript 的 Map（[详见 MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Map)） 会保留插入顺序，这意味着当你遍历一个 Map 对象时，它会按照你插入顺序来进行迭代。而且 Map 还有一个很重要的特性，它的键可以是任意数据类型（Object 的键只能是字符串或Symbol），这很符合缓存的应用场景.

所以实现的思路是：

- 在 get 方法中，如果缓存中存在该键，将其**移到最前面**表示最近使用（删除该 key，再重新 set 即可作为最后插入的最近使用原色）；

- 在 put 方法中，如果已经存在 key，将其**移到最前面**表示最近使用，操作同 get；如果缓存已满，删除**最久未使用**的键，也就是遍历 map 时的第一个 key。

```javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key) {
    if (this.cache.has(key)) {
      // 如果缓存中存在该键，将其移到最前面表示最近使用（提升新鲜度）
      const value = this.cache.get(key);
      this.cache.delete(key);
      this.cache.set(key, value);
      return value;
    }
    return -1; // 如果缓存中不存在该键，返回 -1
  }

  put(key, value) {
    if (this.cache.has(key)) {
      // 如果缓存中已经存在该键，更新其值并移到最前面表示最近使用
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      // 如果缓存已满，删除最久未使用的键（第一个 key 就是最近最少使用，因为 Map 按照插入顺序遍历）
      const oldestKey = this.cache.keys().next().value;
      this.cache.delete(oldestKey);
    }

    // 将新键值对插入到缓存中并移到最前面表示最近使用
    this.cache.set(key, value);
  }
}

// 示例
const lruCache = new LRUCache(2); // 设置容量为2

lruCache.put(1, 1); // 缓存是 {1=1}
lruCache.put(2, 2); // 缓存是 {1=1, 2=2}
console.log(lruCache.get(1)); // 返回 1，缓存是 {2=2, 1=1}
lruCache.put(3, 3); // 删除 key 2，添加 key 3，缓存是 {3=3, 1=1}
console.log(lruCache.get(2)); // 返回 -1 (未找到)，缓存是 {3=3, 1=1}
```
