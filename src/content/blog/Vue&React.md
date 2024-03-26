---
pubDatetime: 2024-03-26T14:10:14.112Z
title: 框架篇 Vue & React
featured: true
draft: false
tags:
  - Vue
  - React
  - 面试
description: Vue 和 React 作为当下最主流的开发框架，是必须要知道其各个细节的。
---

Vue 和 React 作为当下最主流的开发框架，是必须要知道其各个细节的。

## Table of contents

## Vue2 和 Vue3

### 1. 响应式的实现原理

参考：https://febook.hzfe.org/awesome-interview/book1/frame-vue-data-binding

Vue2 通过 `Object.defineProperty`，Vue3 通过 Proxy 来劫持 state 中各个属性的 getter、setter。其中 getter 中主要是通过 Dep 收集依赖这个属性的订阅者 watcher，setter 中则是在属性变化后通知 Dep 收集到的订阅者，派发更新。

1. Dep：实现发布订阅模式的模块。
2. Watcher：订阅更新和触发视图更新的模块。

`Object.defineProperty` 伪代码

```javascript
const dep = new Dep();
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter() {
    // 每次 get 时如果有订阅者则添加订阅
    if (Dep.target) {
      dep.depend();
    }
    return val;
  },
  set: function reactiveSetter(newVal) {
    val = newVal;
    // 每次更新数据之后广播更新
    dep.notify();
  },
});
```

`Proxy` 伪代码

```javascript
return new Proxy(data, {
  get(target, key) {
    const value = Reflect.get(target, key);
    if (typeof value === "object") {
      // 如果是嵌套属性，这里 get 需要递归代理
      return observe(value, callback);
    }
    return value;
  },
  set(target, key, newValue) {
    callback(key, newValue); // 改变属性值，回调
    return Reflect.set(target, key, newValue);
  },
});
```

整体的工作流程就是：属性更新时会触发属性的 setter，setter 中会触发 Dep 的更新，Dep 通知在 getter 中收集到的 watcher 更新，watcher 获取到更新的数据之后触发更新视图。

Vue2 使用的 `Object.defineProperty` 并不能完全劫持所有数据的变化，以下是几种无法正常劫持的变化：

- 无法劫持新创建的属性，为了解决这个问题，Vue 提供了 `Vue.set` 以创建新属性。
- 无法劫持数组的变化，为了解决这个问题，Vue 对数组原生方法进行了劫持。
- 无法劫持利用索引修改数组元素，这个问题同样可以用 `Vue.set` 解决。

Vue 3 中改用 Proxy 实现数据劫持，解决了上面的问题，Vue.set、Vue.delete 等全局方法在 3 中被移除。

Proxy 是 ES6 引入的，不兼容 IE（只有它，看 MDN 兼容 Edge），可以通过 polyfill 来模拟部分 traps，并不完美。

### 2.Diff 算法

虚拟 DOM 的本质是 JavaScript 对象，是 DOM 的抽象简化版本。通过预先操作虚拟 DOM，在某个时机找出和真实 DOM 之间的差异部分并重新渲染，来提升操作真实 DOM 的性能和效率。

为达到这个目的，还需要关注两个问题：什么时候重新渲染，怎么高效选择重新渲染的范围。找出需要重新渲染的范围，就是 Diff 的过程。

React 和 Vue 的 Diff 算法思路基本一致，只对同层节点进行比较，利用唯一标识符 key 对节点进行区分。

React 从根元素开始：

- 不同类型的元素，比如 div 直接变 button 了，直接抛弃旧树，创建新树；
- 相同类型的元素，保留 DOM 节点，比对更新两者的属性；然后递归元素的子元素。

Vue 的 diff 与 React 类似：

1. 只在同一层次进行比较，不进行跨层比较；
2. 不同类型元素，不进行递归比较

![vue-diff](https://user-images.githubusercontent.com/17002181/130326547-7cdcbc06-b400-43d3-89ce-e73934e38bdf.png)

在 diff 子元素的时候，Vue 采用的是双端比较法，设立了四个指针：新列表的 start，end；老列表的 start，end，同时遍历新老虚拟DOM 列表，并采用头尾比较法，四种情况：

1. 新老 start ，指向的是相同节点
2. 新老 end ，指向的是相同节点
3. 老 start 和新 end，指向的是相同节点
4. 老 end 和新 start，指向的是相同节点

上面四种情况，复用节点然后按需更新属性，对应两个指针一次移位；

如果均不满足，会尝试检查新 start 的 key，如果能在旧列表中找到相同 key 的相同类型节点，复用并按需更新属性。如果均不满足，就新增节点。

### 3. Vue3 相比 Vue2 的其它优化

除了响应式原理的更新之外，Vue3 优化了：

- 支持 Composition API，相对于 Vue2 的 options API，组件的代码组织更加灵活，不再受限于特定的格式如 data、methods、computed 等；
- 源码使用 TypeScript 重写，准确的类型定义和类型推断；
- 全局 API 引入 TreeShaking，如 nextTick，通过 import 导入使用，可以做静态分析；

## React

### 1. 事件机制

参考：https://febook.hzfe.org/awesome-interview/book4/frame-react-event-mechanism

如果你在 React 中对一个 div 绑定了 onClick 事件，它不是将事件绑定在真实的 DOM 上的，而是在 document 处监听了所有的事件，当时间发生并且冒泡到 document 时，React 将事件内容封装，交由真正的处理函数运行。这就是 React 自己实现的合成事件。

这么做的用意在于：

- 抹平浏览器之间的差异，简化事件逻辑（如每当表单类型组件的值发生改变时，都会触发 onChange 事件，而 onChange 事件由 change、click、input、keydown、keyup 等原生事件组成）
- 降低在各自组件上创建原生事件的内存开销

### 2. Hooks 相关

参考：https://www.explainthis.io/zh-hans/swe/what-is-react-hook

使用使用 useXXX 开头的函数，使得 funtion 组件也能够使用到 React 的功能和状态。

React hooks 实现的关键：

- 数组
- 不可变数据 immutable

先说数组。React Hooks 的实现都依赖数组。为什么每个 Hook 的文档中都提到一个约束：只能在函数式组件的最顶层使用 Hook？

```javascript
const [number, setNumber] = useState(0);
const [showMore, setShowMore] = useState(false);
```

我们定义了两个 state，但我们传入的都是一个原始值，React 是怎么知道哪个 state 对应到哪个 useState 的呢？在 re-render 的时候，React 又是如何正确的取值的呢？

其实 Hook 背后的实现，就是依赖我们在代码中的调用顺序。

他维护了两个数组，一个存 state[] 值，一个存 setters[] 方法，还维护者一个 cursor，初始化为 0。每当调用一次 useState ，就往 state[] 数组里push入初始值，然后创建一个 setter 方法，push 到 setters 数组，cursor++，返回[state[cursor], setters[cursor]]。我们通过数组结构，拿到 state 值和 setter 函数。

重新渲染的时候，cursor 会被重置为 0，然后从 0 开始在根据 use 的顺序，依次取出之前的 state 和 setter。只要这个顺序没有在 re-render 时发生变化，那么就可以拿到正确的 state 和 setter。

state 并无明确的标识和引用，都依赖我们 use 的顺序作为他在数组中的索引。如果 useState 出现在 if else 或其它语句中，就没法保证他的顺序是稳定的，这样会产生问题。

再说不可变数据。

为什么 Hooks 都要求使用不可变数据？setState 时要求传入是 immutable data，或者是 useEffect 的 dependencies 要求 deps 的变换必须是新的对象，才会触发 effect 副作用？

因为 React 的响应式实现原理，基于【 Object.is 浅比较】来决定当前数据是否改变，进而重新渲染组件。如果是直接修改 Object 的 key，引用地址没有变化，它是无法感知到的。

所以在 React 中 setState 时每次都要传入新的对象，擅用 ... 展开运算符，在操作数组的时候要尤为注意：

- push 和 unshift ，pop 和 shift 都是直接改变原数组，有副作用
- concat 拼接，slice 浅拷贝&截取，filter 过滤，map 遍历都是会返回新的数组，无副作用，维持 immutable。

### 3. 理解 Fiber 架构

参考：https://juejin.cn/post/7077545184807878692?from=search-suggest

React 16 引入 Fiber 架构，可以理解为一个更强大的虚拟 DOM。它的目的，就是为了支持为了支持“可中断渲染”而创建的。

> 在之前的版本中，React 使用递归的方式处理组件树的更新，这种方法一旦开始就不能中断，直到整个组件树都被遍历完。这种机制在处理大量数据或复杂视图时可能导致主线程被阻塞，从而使应用无法及时响应用户的输入或其他高优先级任务。
> Fiber 可以理解为 React 自定义的一个带有链接关系的 DOM 树，每个 Fiber 都代表了一个工作单元，React 可以在处理任何 Fiber 之前判断是否有足够的时间完成该工作，并在必要时中断和恢复工作。

传统的虚拟 DOM 是多叉树形结构，每个节点内保留它自身属性和他的 children，在 diff 操作中，对树的遍历操作从根节点开始，递归查找子节点，这个过程不可中断，且不可逆。如果这个子树非常庞大，就会造成 UI 线程阻塞。

Fiber 的实现，扩展了虚拟 DOM，使用链表取代树，将虚拟 DOM 进行连接。每个 fiber 节点有三个指针，分别指向第一个子节点，和下一个兄弟节点，再通过 return parent 指向当前节点的父节点。节点内同时新增了一些进度保存的属性，在 render 发生中断后，保留下当前节点的索引，当前节点又持有父节点的指针，就可以继续从中断节点恢复工作。

注意我们所说的可中断恢复，仅限于框架的【 render 阶段】，也就是虚拟 DOM 的 diff 和变更，最终 【commit 阶段】操作真实 DOM 渲染 UI 是不能终端的。

fiber 的不阻塞更新，实际上是通过这种中断和恢复的能力，把组件的 render切分成多个工作分片，每个分片完成后就会让出主线程，去渲染其它优先级更高的任务。

![fiber-rrender](@assets/images/vue-react/fiber-render.webp)

对出让主线程的实现，window 有 requestIdleCallback 可以支持在空闲时间执行任务，同时还可以在回调的入参 deadline.timeRemaining() 获取当下可以执行的预估时间。不过 由于这个 API 对 callback 执行时机并不完全，而且在 Safari 上不兼容，React fiber 是使用的自己实现的 API。

## Vue 和 React

### 对二者的理解

使用框架，最大的收益就是开发者能够以现代化组件的方式进行开发，在组件的【状态】发生变化时视图自动更新，避免了频繁的手动对 DOM 的操作。两者都最大的区别，在于对于【响应式】的设计和实现上。

Vue 的状态和视图之间，存在这精确的对应关系，在响应式数据构建的时候，就对数据的 get、set 操作做了劫持，把当前发起 get 对象添加到 watcher 队列中。在 set 更新数据时，就可以知道具体需要更新哪些视图，重新渲染。

React 中组件的状态是 immutable 不可变的，通过 setState 去更新状态也没有修改原来的内存变量，而是开辟了一个新的内存。在 setState 更新状态之后，React 会递归遍历当前组件及其所有的子组件，重新渲染整个组件子树。这个计算量相对而言是庞大的，这也就是为什么在优化 React 时，我们需要借助 memo 和 useCallback 等方式来明确告知 React 组件在什么情况下需要重新渲染，尽量减少不必要的渲染。Fiber 架构也是为了解决这个问题。

Vue 在渲染过程中自动完成依赖绑定，取决于他的模板语法，同时维护状态和视图的依赖关系也会带来性能上的消耗。React 直接扩展了 JSX 来实现视图，结合函数式组件，又带来了更好的自由度。

所以从直观上，Vue 看起来更加自动化，写起来也更加容易，框架本身替你做了绝大多数工作。而 React 通常伴随着一些心智上的负担，虽然通过 hooks 和 api 提供了很多能力，但新手往往不清楚到底怎么做才是最好的。
