---
pubDatetime: 2024-02-20T11:10:00Z
title: JavaScript 原型、原型链究竟在干啥？
featured: true
draft: false
tags:
  - 前端
  - JavaScript
description: JavaScript 的 prototype 究竟是个啥？我这种半路出家的前端，在网上看了很多说法，也问了 ChatGPT，看完还是云里雾里，迷迷糊糊，成了绕不过去的一个坎。
---

JavaScript 的 prototype 究竟是个啥？我这种半路出家的前端，在网上看了很多说法，也问了 ChatGPT，看完还是云里雾里，迷迷糊糊，成了绕不过去的一个坎。

## Table of contents

## 一段代码，测试一下

```javascript
function Foo() {
  getName = function () {
    console.log(1);
  };
  console.log(this);
  return this;
}

Foo.getName = function () {
  console.log(2);
};

Foo.prototype.getName = function () {
  console.log(3);
};

var getName = function () {
  console.log(4);
};

function getName() {
  console.log(5);
}

Foo.getName(); // 2
getName(); // 4
Foo().getName(); // 1
getName(); // 1
new Foo().getName(); // 3
```

正确的输出已经写注释上了，如果你也不明所以，那么不妨继续读下去。

## First of all，为什么会有这套原型系统？

这里我们不深入探究 JavaScript 的发展历史，以及关于他是否真正的面向对象，这里感兴趣可以去参考 winter 在《重学前端》中关于这块的章节。简而言之，早期的 JavaScript 并没有 class 关键词，来像 Java 一样直观的定义类，extends 关键字来表示类的继承关系（ES6 已加入），原型系统的设计得以让 JavaScript 也可以实现基于类的面向对象效果。
也就是说，现在我们可以通过两种方式来实现类，一是 class 方式实现，这个是 Java 开发者都很熟悉的了：

```javascript
class Range {
  // 构造函数
  constructor(from, to) {
    this.from = from;
    this.to = to;
  }

  // 实例方法
  toString() {
    return `(${this.from}...${this.to})`;
  }

  // 静态方法
  static getDesc(name) {
    return "Range " + name;
  }
}
```

但实质上，class 关键字是语法糖，底层实现还是原型系统，那么使用原型系统来实现这个类：

```javascript
// 构造函数
function Range(from, to) {
  this.from = from;
  this.to = to;
}

Range.prototype.toString = function () {
  return `(${this.from}...${this.to})`;
};

Range.getDesc = function (name) {
  return "Range " + name;
};

// 创建实例
const r = new Range(1, 3);
r.toString(); // (1...3)
Range.getDesc("r"); // Range r
```

建立这层一一对应关系，就基本上可以理解原型的基础了。
需要注意的是，创建实例时，必须要使用 new 关键字，否则就成了一次普通的函数调用。

## 关键概念

![prototype-overall](@assets/images/javascript-prototype/overall.jpg)

### 构造函数

构造函数是专门用户初始化新对象的函数，概念上和 Java 中类的构造函数基本一致，可能比较迷惑的点在于，构造函数在 JavaScript 中是以一个独立函数的形式存在的，他和普通函数长的一样（构造函数的首字母大写，实际上是一种命名约定），但如果做构造函数用，则**必须使用 new 关键字来调用**，这样会自动创建一个新对象，构造函数中的 this 就指代这个新对象。默认构造函数返回新对象。

### 类和原型

理解原型，离不开类和对象的概念。
对象的创建过程，如果理解为“比猫画虎”，那虎就是猫的原型。对照到语言中，可以概括为：如果我创建了一个实例 r，那 r 的原型就是 Range.prototype。原型本身也是对象。我们如果验证这一点？
![prototype-chain](@assets/images/javascript-prototype/getPrototypeOf.png)
在代码块 Code-3 的基础上，我们 Object.getPrototypeOf 查 r 的原型，会发现有我们定义的 toString，还有一个默认的 constructor 属性，而这个属性正式指向 Range 构造函数本身。在浏览器环境下默认为对象提供了 `__proto__`来取得对象的原型，不过这个并不在 ECMAScript 规范中。
原型 prototype 和 构造函数 constructor 就这样形成了互相指认。

### new 关键字

根据前面构造函数的定义，函数必须是通过 new 关键字来调用，才会被作为构造函数，那么 new 关键字里究竟做了什么？可以通过自己实现一个 new 的方式来理解。

```javascript
function myNew(fn, ...args) {
  // 1. 创建了一个空对象
  let obj = {};
  // 2. 将空对象的原型，指向构造函数的原型
  Object.setPrototypeOf(obj, fn.prototype);
  // 3. 将空对象作为构造函数的上下文（改变 this 指向）
  let result = fn.apply(obj, args);
  // 4. 对构造函数的返回值进行处理（引用类型直接返回，基本类型抛弃，返回新对象）
  return result instanceof Object ? result : obj;
}
```

new 默认是有返回值的，如果构造函数内有写 return，且 return 结果是引用类型，那就返回构造函数 return 的对象，否则返回新创建的对象。所以在一般情况下，我们不会在一个明知是构造函数的函数中写 return 逻辑，这也是面向对象编程的一个共识。

## 通过 Object.create 看原型链

Object.create 用于创建一个新对象，使用第一个参数作为新对象的原型。

```javascript
let o1 = Object.create({ x: 1, y: 2 }); // o1 集成属性 x 和 y
o1.x + o1.y; // => 3
```

补充：Object.create 的原理

1. 创建一个空的构造函数 F
2. 将其原型设置为传入的 proto 对象
3. 通过 new F() 创建一个新对象，这个对象的原型链指向了 proto。

```javascript
function myCreate(proto) {
  function F() {}
  F.prototype = proto;
  return new F();
}
```

（当然这里为了便利，只考虑了 create 第一个参数）
从上面不难看出，其实 Object.create 其实就是一个简单版本的继承：创建了一个匿名类的实例，该实例继承自入参对象的原型。

实例观察一下，我们在 Code-3 的基础上，再通过 Object.create 创建一个对象 r2
![image.png](@assets/images/javascript-prototype/create.png)
查看 r2 的原型，就是 r 对象。不难看出

- r2 的原型是 r
- r 的原型是 Range.prototype
- Range.prototype 再往上一级的原型是 Object.prototype
- Object.prototype 是原型链的最顶端，他的原型是 null

这样就构成了一条原型链：
`r2.__proto__ -> r.__proto__ -> Range.prototype.__proto__ -> Object.prototype.__proto__ -> null`
根据原型系统的规则，属性的查找会顺着这条原型链一直向上，直至 null。

所以，Code-3 Lin3-17，当我们在调用 `r.toString()`的时候，会发现 r 对象本身并没有这个方法，那么根据原型链规则，会向原型链的上一级去查找，也就是前面示例提到的 Range.prototype，在这里找到了 toString 方法，执行该方法。如果这里没有自己定义 toString，那么会追踪到 Object.prototype.toString ，那里有一个默认的方法实现。

## 总结

回到最初的问题，其实这段代码除了原型和原型链，还考察了作用域相关的知识，整个过程见注释：

```javascript
function Foo() {
  getName = function () {
    // 注意：这里 getName 会挂载到全局上下文
    console.log(1);
  };
  console.log(this); // 这里 this 取决于函数的调用
  return this;
}

Foo.getName = function () {
  // 这里等同于定义静态方法
  console.log(2);
};

// 为 Foo 新增一个原型方法
Foo.prototype.getName = function () {
  console.log(3);
};

// var 声明的变量，会被提升到作用域最顶部，这是作用域是全局，
// 相当于 var getName = undefined ... getName = xxxx，赋值的函数表达式，不具名函数，执行时机不变
var getName = function () {
  console.log(4);
};

// 函数声明，会被提升到作用域最顶部，这样可以实现在函数声明之前调用函数
// 这里具名函数声明提升到顶部之后，会被上面的 getName = function xxx 覆盖掉，永远不可访问
function getName() {
  console.log(5);
}

Foo.getName(); // 2，直接调用静态方法
getName(); // 4 ，等同于 this.getName()，调全局方法
Foo().getName(); // 1，先按普通函数执行了 Foo()，给全局挂载了一个新的 getName，返回的 this 也是全局上下文
getName(); // 1，上一步修改了全局上下文的 getName
new Foo().getName(); // 3，这里有 new 关键字，按构造函数调用了 Foo() 会创建一个新对象，此时内部 this 指向的就是这个新对象，然后在该对象的属性中查找 getName，如果没有就往其原型上找，此时找到 3
```

## 扩展：继承的实现

从上一节的内容，Object.create 其实就是一个简单版本的继承：创建了一个匿名类的实例，该实例继承自入参对象的原型。如果我们想要实现一个具体的类的继承呢？
如果使用 ES6 的新语法，那直接使用 extends 关键字就可以，和 Java 是一样的。

```javascript
class Range {
  // 构造函数
  constructor(from, to) {
    this.from = from;
    this.to = to;
  }

  // 示例方法
  toString() {
    return `(${this.from}...${this.to})`;
  }
}

class LengthRange extends Range {
  constructor(from, to, unit) {
    super(from, to);
    this.unit = unit;
  }
}
```

如何通过原型的方式实现继承呢？

```javascript
// 构造函数
function Range(from, to) {
  this.from = from;
  this.to = to;
}

Range.prototype.toString = function () {
  return `(${this.from}...${this.to})`;
};

// 1.原型式继承
function LengthRange(from, to, unit) {
  Range.call(this, from, to); // 类似 super，子类取得父类的实例属性和方法
  this.unit = unit;
}
// 2.父类实例作为子类的原型
LengthRange.prototype = Object.create(Range.prototype);
// 3.修复构造函数的指向
LengthRange.prototype.constructor = LengthRange;
```

1. 第一步，子类构造函数，通过调用父类构造函数并传入当前新对象的 this，来获取和绑定父类的实例属性和方法；
2. 到这已经实现了继承的基本，但是还是没有组成原型链，子类是无法获取到父类的原型方法的，比如上面代码中的 toString 方法，如果通过 LengthRange 的实例去调用 toString，拿到的会是 Object 的 toString；

![image.png](@assets/images/javascript-prototype/extend1.png)

所以第二步是通过 Object.create 来创建一个父类的原型对象赋值给子类的原型，达到父子类原型链接的目的。

![image.png](@assets/images/javascript-prototype/extend2.png)

3. 到这里就基本完成了继承。但是因为我们整个重新赋值了子类的原型 LengthRange.prototype，带来两个问题：
   1. constructor 默认是在原型上的构造函数引用，重新赋值 prototype 之后会丢失，所以就有了第三步的修复构造函数指向；
   2. 对子类的定义原型方法如 LengthRange.prototype.xxxx 一定要写在继承代码之后，否则就会丢失；
4. 补充内容：通过以上步骤，我们实现了寄生组合式的继承，达到了子类继承父类的效果，完成了原型链的建立。但是不难发现，我们没处理静态方法。也就是说如果我们通过 LengthRange 是无法获取到 Range.xxx 静态方法的。因为静态方法本身与对象实例无关，也就意味着在静态方法内无法取得当前实例的 this，所以我们一般不需要去继承，除非你的业务场景有这种特殊的需要。如果想要实现静态方法的继承，需要加上一句：`Object.setPrototypeOf(LengthRange, Range);`。这里可能会有点迷惑，构造函数本身也是有原型的，默认情况加`Object.getPrototypeOf(LengthRange)` 就是 Function，我们手动 set 之后就把他的原型改变成了 Range 函数，而 Range 的静态方法就是绑定在构造函数本身，所以这样就可以通过 LengthRange.xxx 调用静态方法了。这里理解时一定要和 `LengthRange.prototype` 原型对象区分开。
