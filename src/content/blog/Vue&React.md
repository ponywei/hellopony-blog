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

Vue2 通过 `Object.defineProperty`，Vue3 通过 Proxy 来劫持 state 中各个属性的 getter、setter。其中 getter 中主要是通过 Dep 收集依赖这个属性的订阅者 watcher，setter 中则是在属性变化后

通知 Dep 收集到的订阅者，派发更新。

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

Refer：[Link](https://febook.hzfe.org/awesome-interview/book1/frame-vue-data-binding)

### 2. Vue3 的 ref 和 reactive

#### ref(0) / ref({})

- 可以传入原始数据类型，如数字或者 boolean 等，把它们包装成一个响应式的 RefImpl 对象，通过 getter setter 来劫持对 value 属性的操作，所以在使用时需要通过 x.value 来获取或更新响应式的值；
- 当然也可以传入对象，如果 ref({…}) 那么效果和 reactive({…})是一样的，源码的处理逻辑是：同 reactive({…}) 生成一个对象 Proxy 代理对象，然后赋值给 value，即 ref.value=proxy;
- 通过 Dep 自身管理依赖，同 Vue2；
- watch 默认只观察 ref.value，如果需要深度监听需要传入 { deep: true }；官方推荐使用 watchEffect，响应式地追踪其内部依赖，并在依赖更改时重新执行回调。(watchEffect 有 React Hook 那味儿了，不过本质上还是有区别，Vue 的 watchEffect 依赖也是自动收集的，React 需要手动声明，或者借助 lint 工具检查。)

#### reactive({…})

- 无法操作原始类型，只能传入一个对象，它通过 Proxy 代理，劫持 get set 方法实现响应式，返回的是一个新的 proxy 对象，后续读取属性均需要通过 proxy 对象；
- 通过全局的一个 WeakMap 来管理依赖；
- watch 默认支持深度监听。

以上：按需取用，首选 ref。

Refer：

- [响应式基础](https://cn.vuejs.org/guide/essentials/reactivity-fundamentals.html)
- [深入响应式系统](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html)

### 3.Diff 算法

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

### 4. Vue3 相比 Vue2 的其它优化

除了响应式原理的更新之外，Vue3 优化了：

- 支持 Composition API，相对于 Vue2 的 options API，组件的代码组织更加灵活，不再受限于特定的格式如 data、methods、computed 等；
- 源码使用 TypeScript 重写，准确的类型定义和类型推断；
- 全局 API 引入 TreeShaking，如 nextTick，通过 import 导入使用，可以做静态分析；

### 5. 组合式 API 与组合函数

Vue 组合式 API 的优势，按照业务自行组织代码结构只是一方面，最重要的一点在于【逻辑复用】，可以是有状态的，就是 VueUse 做的那样，一个类似自定义 React Hooks 的库，命名约定也是 useXXXX，在 Vue 里叫做组合函数。

`<script setup>` 是语法糖，只能适用于单文件组件 SFC，等同于 setup() 函数，编译时会把里面的代码转成 setup() 函数的内容。所以，setup 里的代码会在每次组件实例被创建时都会执行。普通 script 标签里的代码只会在组件首次引入时执行。这里是很容易产生误解的地方。

```vue
<script setup>
const count = ref(0);
</script>
```

```vue
<script>
export default {
	setup() {
		const count = ref(0);
    // 返回值会暴露给模板和其他的选项式 API 钩子
    return { count };
  },
}
</scirpt>
```

为什么 VueUse 组合函数 一定要在 setup 中使用？因为这样才能：

1. 将生命周期钩子注册到该组件实例上
2. 将计算属性和监听器注册到该组件实例上，以便在该组件被卸载时停止监听，避免内存泄漏。

Refer:

- [组合式函数](https://cn.vuejs.org/guide/reusability/composables.html)
- [和 React Hooks 相比](https://cn.vuejs.org/guide/extras/composition-api-faq.html#comparison-with-react-hooks)

## React

### 1. 事件机制

如果你在 React 中对一个 div 绑定了 onClick 事件，它不是将事件绑定在真实的 DOM 上的，而是在 document 处监听了所有的事件，当时间发生并且冒泡到 document 时，React 将事件内容封装，交由真正的处理函数运行。这就是 React 自己实现的合成事件。

这么做的用意在于：

- 抹平浏览器之间的差异，简化事件逻辑（如每当表单类型组件的值发生改变时，都会触发 onChange 事件，而 onChange 事件由 change、click、input、keydown、keyup 等原生事件组成）
- 降低在各自组件上创建原生事件的内存开销

Refer：[React事件机制](https://febook.hzfe.org/awesome-interview/book4/frame-react-event-mechanism)

### 2. Hooks 相关

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

Refer：[什么是 React Hook](https://www.explainthis.io/zh-hans/swe/what-is-react-hook)

### 3. 关于 React 闭包陷阱

闭包陷阱的产生，就是 state 发生变化了，但是 useEffect 的更新函数没有重新执行，导致更新函数引用的 state还是之前的旧值。

1. 正确传入 deps，如果依赖有变化，重新执行 updater 函数。不传 deps 每次 re-render 都重新执行，传[]空数组只在第一次执行；
2. 注意添加 return 清除函数，消除影响。

Refer：[从根本上解决闭包陷阱](https://zhuanlan.zhihu.com/p/509036942)

延伸：如何通过React Hooks 实现 setInterval 定时执行功能？

详情参见 Dan 博客原文 [Making setInterval Declarative with React Hooks](https://overreacted.io/making-setinterval-declarative-with-react-hooks/)

第一次尝试：

```javascript
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => {
      console.log("Interval ", id);
      clearInterval(id);
    };
  }); // deps 关键

  return <h1>{count}</h1>;
}
```

这个问题，让人迷惑的点在于，看似是实现了，但是没有传 deps 每次 setCount 引起的 re-render 组件都会重新渲染，进而引起的是 clearInterval 和 setInterval 重新执行，实际每次都不是同一个 interval。我们可以通过在 return 清理函数中打印 id 查看（注意，每次 re-render 会先执行清理函数，再重新执行 effect 函数）。

那如果我们加上 deps [] 空数组，显然只会执行一次，渲染 1 就停止了。

```javascript
useEffect(() => {
  let id = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => {
    console.log("Interval ", id);
    clearInterval(id);
  };
}, []); // deps 关键
```

而且这样带来的问题是：我们的 effect 函数里依赖了 count，他是响应式状态数据，但是我们没有在 deps 里声明它。如果项目配置了eslint-plugin-react-hooks 规则的话，这样是会被卡控的。也就是说，这样做是被禁止的！

那如何能解除对 state 的依赖？

其实可以借助 setState() 的 updater 函数，实现[基于前值更新 state](https://react.dev/reference/react/useState#updating-state-based-on-the-previous-state)：

```javascript
useEffect(() => {
  let id = setInterval(() => {
    setCount(count => count + 1); // 关键：n => n + 1
  }, 1000);

  return () => {
    console.log("Interval ", id);
    clearInterval(id);
  };
}, []); // deps 关键
```

这样在第一次渲染之后，effect 函数便不会再被执行，interval 实例实现复用，setCount 每次被调用时，入参是上一次的值，再此基础上计算返回新值，解除了对 count state 的依赖。

所以在 Dan 的原文里，又增加了一个复杂度：setInterval 的 delay 可以手动输入，这就要求组件要接收一个动态的 props，同时应用于 effect 中。我们没有办法直接解除对 props 的依赖。

本质上，我们不希望每次重新渲染都要再定义一个 Interval 出来，所以可以把 setInterval 的 callback 抽取出来，因为是 callback 里依赖了 count 状态，这样解耦 count 和 setInterval。

完美的解决办法是：自定义 Hook + useRef

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}

function useInterval(callback, delay) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

重点：

- 利用 useRef 不重新渲染的特性，缓存 callback，只有外部传入 callback 更新的时候重新赋值 .current = callback
- 首次，或者外部传入 delay 变化时，执行 setInterval，回调函数传入 ref.current

### 4. 理解 Fiber 架构

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

### 5. 常用 Hooks 索引

<details>   <summary>点击展开/折叠</summary>
  
#### [useState](https://react.dev/reference/react/useState) 重中之重

const [state, setState] = useState(initializer)

useState 返回值是一个数组，通过数组解构可以取得一个只读的 state 和一个 set 函数【为什么是数组？因为解构的时候我们可以自己指定名字，如果是对象解构，必须显示的指定 key 然后才能设置别名，妙~】

setXXX(updater)：

- 异步更新，next render 才生效，set 之后立马获取值会取到 old value；

- 批量更新，等待所有的 event handler 都 set 之后才更新，避免重复；可以通过 flushSync 强制提前更新；

  - 这里对批量更新可能会有误区，并不是多个 set 合并一个，而是维护一个状态更新队列，下次渲染时 next render 时，遍历执行队列里的任务，达到整合 set 的效果，详见 →

  [Queueing a Series of State Updates – React](https://react.dev/learn/queueing-a-series-of-state-updates)

- 如果 set 的新值和旧值相同（[Object.is](http://Object.is) 判断），会跳过 re-render；

注意上面标蓝的 ：初始化和更新，可以传入值，也是可以传入函数的，三种情况：

1. setNumber(number + 1)，这会取 number 的旧值 + 1，然后入队列的指令是“替换为 1”；
2. setNumber(n => n + 1)，这里入队列的指令是箭头函数 n => n + 1，next render 时取旧值计算并返回新值；
3. 如果你要报错的 state 是一个函数，❎setFn(fn); 这是一个错误案例， fn 是一个函数，但直接传函数引用会被当做更新函数立即执行掉，

✅setFn(() ⇒ fn) 这样传入一个箭头函数，该函数执行后会返回 fn 函数，存入任务队列就是 fn 函数了。

为什么初始化 or 更新函数会执行两次？

注意，这仅仅会发生在开发环境下的 Strict mode，不影响生产环境。

因为 React 故意这样做的，初始化和更新函数必须是**纯函数**，执行两次就是为了在有副作用的时候暴露在开发过程中。

```jsx
setTodos(prevTodos => {
  // 🚩 Mistake: mutating state
  prevTodos.push(createTodo());
});

setTodos(prevTodos => {
  // ✅ Correct: replacing with new state
  return [...prevTodos, createTodo()];
});
```

#### [useRef](https://react.dev/reference/react/useRef) 有点多功能了

1. let ref = useRef(0) 返回一个 ref 对象，只有 current 属性，可以读写；

   - re-render 时不会重新创建对象，它是跨 render 共享的；
   - 改变值不会引起 re-render，因此不适合存 UI 渲染依赖的数据；
   - 重复创建多个 Component 时，它的值是 local 的，非 shared；（有待验证）

   ```jsx
   export default function Counter() {
     let ref = useRef(0);

     function handleClick() {
       ref.current = ref.current + 1;
       alert("You clicked " + ref.current + " times!");
     }

     return <button onClick={handleClick}>Click me!</button>;
   }
   ```

2. 通过 ref 操作 DOM

```jsx
import { useRef } from "react";

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>Focus the input</button>
    </>
  );
}
```

但是要注意的是：这里是直接操作原生组件，如果是自定义组件 ref，需要包裹 forwardRef 传入 ref，详见文档。

1. 基于特性 1，用于避免重复创建实例

```jsx
function Video() {
  const playerRef = useRef(null);
  if (playerRef.current === null) {
    playerRef.current = new VideoPlayer();
  }
   // ...
```

#### useEffect 重器

useEffect(setup, dependencies?)

- setup 函数 return () ⇒ { 清理代码 }，用于重新 re-render 之前的清理工作，数据重置啥的；
- dependencies 是个数组
  - 传入空数组，仅第一个 render 初始渲染的时候执行；
  - 完全不传该参数，render 和每次 re-render 都会执行；
  - 传入数组，有依赖项，初始渲染以及依赖项有变更时候执行；
  - 如果 effect 内部使用了其它的 Reactive value，必须将他们声明在依赖项中，否则 lint 会提示错误；
  - Reactive value 包含所有在组件内部定义的变量，不需要的话就移动到组件外部定义。

注意：如果在 Effect 内部更新 state（依赖项有 state） 可能会造成循环更新问题，需要借助 setState 的 updater 函数参数(上一次的旧值）来进行计算，排除 state 依赖：

```jsx
const [count, setCount] = useState(0);

useEffect(() => {
  const intervalId = setInterval(() => {
    // setCount(count + 1); // 你想要每秒递增该计数器...这样需要声明 count 为依赖项
    setCount(c => c + 1); // ✅ 传递一个 state 更新器
  }, 1000);
  return () => clearInterval(intervalId);
}, []); // ✅现在 count 不是一个依赖项

return <h1>{count}</h1>;
```

#### useContext 透传数据

useContext + createContext + xxxxContext.Provider 用来跨越组件层级的【读取和订阅】 数据。注意包裹。如果传入 setState 方法就也可以在子层级修改数据。

[Passing Data Deeply with Context – React](https://react.dev/learn/passing-data-deeply-with-context)

#### useCallback 缓存函数定义

useCallback 用来【在 re-render 时】缓存函数的**定义，直至 dependencies 发生变化。**

同 useEffect 一样第二个参数 [deps] 传入响应式依赖。如果re-render 时依赖项没有变化，返回初始化的函数，否则重新创建函数返回。

**在默认情况下，当一个组件发生re-render 时，React 会递归的 re-render 它的所有子组件。**

memo(component) 用来缓存一个组件的定义，memo 之后如果组件的 props 未发生变化，则该组件不会 re-render。

但是如果这个组件的入参有 function，这个函数定义在父组件，通过 props 传入。如果是直接**`function () {}` or `() => {}` 这种函数定义，那么实际上每次父组件 re-render 时走到这里都会重新创建函数，那 props 传入的 function 永远都是新的，导致 memo 组件无效。**

此时就需要 useCallback + memo

<aside> 💡 关于响应式（数据和逻辑）的定义 https://react.dev/learn/separating-events-from-effects#reactive-values-and-reactive-logic

响应式数据

- props
- state
- 所有定义在组件体内的 variables

响应式逻辑

- effects 内的逻辑是响应式的。如果 effect 里读取了响应式数据，必须要声明为依赖，这样在 re-render 时 React 会用新的 value 重新执行 effect 逻辑。
- event handlers 内的逻辑是非响应式的，只在用户交互时响应执行。

我的理解：一个函数是否是 reactive 并不是定义时决定的，如果是作为 event handler 那就是非响应式的，如果在 effect 内调用了，那他就是响应式的，需要声明成依赖，进而导致每次 re-render 都会执行函数调用产生新对象，再进而触发重新执行 effect。这显然不是想要的，所以这时候函数就应该使用 useCallback 包裹了。而 useCallback 也是有 dependencies 的，这样就形成了一个链式的依赖响应： useCallback deps 变化 → 函数重新创建执行 → useEffect deps 变化 → effect logic 重新执行

</aside>

#### useMemo 缓存函数结果

useMemo 用来缓存函数的**执行结果**。长相和 memo 接近，用法其实是和 useCallBack 接近，注意区别。

直到 deps 变化，才重新执行函数返回结果，否则返回上次执行的结果。

PS：如果用 useMemo 包裹的函数，return 的又是一个函数，那么可以理解为实际上你需要缓存的是一个函数，可以直接用 useCallback，这样还少些一层 return () ⇒ {…}，其它都是一模一样的。

用上面学的一些 hook 把 todolist 优化了一下，效果显著（之前通过 log 来，实际体验不出来差别，数据量很小）：

- 减少不必要的 re-render，之前 input 输入整个列表都会重新渲染，列表项改动也是整个列表重新渲染
- 父子组件的动静数据要分清，比如 done 的切换，之前的逻辑会有 check 不刷新的问题，props 的数据要在子组件改动，就拿下来搞 自己的 state，这样渲染也合理，改动再通过回调函数传回去

#### useReducer 加强版的 useState

useReducer 用来给组件增加一个 reducer（减速器），说了等于没说。这个稍微有一点绕。贴个代码加强理解。

- code

  ```jsx
  function reducer(state, action) {
    if (action.type === "incremented_age") {
      return {
        age: state.age + 1,
      };
    }
    throw Error("Unknown action.");
  }

  export default function Counter() {
    const [state, dispatch] = useReducer(reducer, { age: 42 });

    return (
      <>
        <button
          onClick={() => {
            dispatch({ type: "incremented_age" });
          }}
        >
          Increment age
        </button>
        <p>Hello! You are {state.age}.</p>
      </>
    );
  }
  ```

其实就是一个加强版的 useState，多了个 reducer 函数来处理多种 action，然后返回新的 state

作用是一致的：

- 如果你的 state 更新逻辑很简单，useState 就够了；直接更新即可；如果是有很多处不同的操作都要响应 state 的变更，useReducer 可以更好的分离开事件响应过程和 state 更新逻辑，结构也更清晰；
- reducer 本身是一个纯函数，不依赖组件和任何其它外部数据，可以更好的单测。

#### **useImperativeHandle 暴露方法**

**`useImperativeHandle` 要结合 useRef 和 forwardRef 一起使用，目的是暴露【自定义组件】的【自定义方法】。**

先前学了，我们可以通过 useRef 拿到 DOM 节点进行操作如 input 的 focus 等；

如果是需要操作自定义组件内的 dom，那自定义组件还需要 forwardRef 包一下，向父级组件透传 ref 的绑定（入参里的 ref，再 set 到 dom 元素的 ref={ref}），关键作用就是 **Exposing DOM**。

如果是想暴露自定义方法，就通过 useImperativeHandle 来接收入参的ref，然后在二参 createHandler 中返回一个对象，里面定义想要暴露的方法。如果要在这些方法里操作当前的 DOM，直接再通过新的 useRef 即可。

注意它的三参 deps 依赖项，内部使用到的响应式数据都需要声明依赖。同 useEffect 一样，如果 deps 有变或者[]缺省，二参的 createHandler 会重新执行，绑定新的 ref。

```jsx
const MyInput = forwardRef(function MyInput(props, ref) {
  const inputRef = useRef(null);

  useImperativeHandle(
    ref,
    () => {
      return {
        focus() {
          inputRef.current.focus();
        },
        scrollIntoView() {
          inputRef.current.scrollIntoView();
        },
      };
    },
    []
  );

  return <input {...props} ref={inputRef} />;
});

export default MyInput;
```

#### useLayoutEffect 阻塞式的 useEffect

`useLayoutEffect` 与 `useEffect` 的作用基本一致，官方文档说，它是一个【在浏览器重绘到屏幕之前调用】版本的 useEffect。

也就是外部经常提到的同步和异步，官方的解释是【是否阻塞浏览器重绘】。

```jsx
const [count, setCount] = useState(100);
const ref = useRef(null);

let i = 1;
useLayoutEffect(() => {
  console.log(ref.current.textContent);
  while (i < 100000000) {
    // 模拟阻塞，可见闪烁
    i++;
  }
  if (count === 0) {
    // 这里很关键，否则会循环 re-render
    setCount(10 + Math.random() * 200);
  }
}, [count]);

return (
  <div onClick={() => setCount(0)} ref={ref}>
    {count}
  </div>
);
```

如上示例，div 默认显示 100，点击后状态改为 0：

`useEffect` 在点击后，count 变化，会先完成视图 re-render 渲染成 0，然后引起 useEffect 重新执行，while 循环之后把 count 改成了一个随机数，表现出明显的闪烁（但是很合理）。

`useLayoutEffect` 逻辑是一致的，但是它会在视图渲染成 0 之前阻塞主线程（到这其实 DOM 已经完成更改重排，可以获取到最新的样式，只是没有 repaint 到屏幕上），完成 effect 里的逻辑，表现是没有闪烁。但是如果录制性能表现，会发现每次点击后的主线程阻塞。它的效果等同于 `componentDidMount` 和 `componentDidUpdate` 两个生命周期方法。

可以给 div 加上 ref 打印 textContent 会发现，两个方法的 DOM 都会先改变成 0 的。

#### **useSyncExternalStore**

用来订阅外部 store 的 hook。

看的不是很懂，大概意思就是在 React 和外部（可能是其他的状态管理，或者浏览器 API）之间进行衔接。

#### **useTransition 延迟更新函数**

以一种不阻塞 UI 的方式更新 state。

```
const [isPending, startTransition] = useTransition();
```

isPending 标识是否还在执行中；

`startTransition(scope)` 函数，入参也是一个函数，其内部就是想做为一个 transition 的状态更新逻辑。React 执行到这里时，会立即执行scope 函数，同步的把里面的 state 变化都标记为 transition，然后它就可以被其它 state 操作打断，这就是不阻塞 UI 的关键。

场景：如果有一个 tab 页数据量比较大，切换 tab 操作使用 startTransition，那么即便 tab 没有渲染出来，用户也可以切换到其它 tab，而不是什么也坐不了，傻傻的等着。

注意，还有个 api `startTransition` ，他不是 hook，可以用在组件外部，原理是一样的，只是没有 isPending 标识了。

#### **useDeferredValue 延迟更新值**

**`useDeferredValue` 和 `useTransition` 接近，都是为了不阻塞 UI 的更新。**不同的是，前者包裹的是一个 state，所以 defer 的只是一个 value，后者的 transition 可以是一个或多个 state 的更新动作组合。

**`useDeferredValue`** 的流程是：当收到一个不同的值时，会先用旧值完成当前的 render，在后台使用新值发起一个 re-render 调度。这个后台 re-render 是可以被打断的。也就是说，如果有其它操作更新了值，React 还会重启 re-render。

```jsx
const [query, setQuery] = useState("");
const deferredQuery = useDeferredValue(query);
return (
  <>
    <label>
      Search albums:
      <input value={query} onChange={e => setQuery(e.target.value)} />
    </label>
    <Suspense fallback={<h2>Loading...</h2>}>
      <SearchResults query={deferredQuery} />
    </Suspense>
  </>
);
```

场景：input + 联想词列表搜索；默认输入值变化，组件会重新渲染

1. 把输入变化 render 到输入框（这个很关键，输入值变化组件也会重新渲染，当次 render 如果耗时过长，直观感觉会卡顿）
2. 把新值传入 SearchResults 组件，用于搜索结果列表渲染

步骤 2 明显会很耗时，我们通过 useDeferredValue 创建一个 query 的 defer 引用，传给子组件。

第一次渲染时，deferredQuery 返回的值跟传入是一样的；query 发生更新后的渲染，React 会先尝试使用上一次的值完成渲染，然后 使用新值发起一个 re-render。

这样表现出至少三次渲染，第二次就是把 input 的值重新渲染到了输入框，第三次才是搜索列表的更新。 </details>

## Vue 和 React

### 对二者的理解

使用框架，最大的收益就是开发者能够以现代化组件的方式进行开发，在组件的【状态】发生变化时视图自动更新，避免了频繁的手动对 DOM 的操作。两者都最大的区别，在于对于【响应式】的设计和实现上。

Vue 的状态和视图之间，存在这精确的对应关系，在响应式数据构建的时候，就对数据的 get、set 操作做了劫持，把当前发起 get 对象添加到 watcher 队列中。在 set 更新数据时，就可以知道具体需要更新哪些视图，重新渲染。

React 中组件的状态是 immutable 不可变的，通过 setState 去更新状态也没有修改原来的内存变量，而是开辟了一个新的内存。在 setState 更新状态之后，React 会递归遍历当前组件及其所有的子组件，重新渲染整个组件子树。这个计算量相对而言是庞大的，这也就是为什么在优化 React 时，我们需要借助 memo 和 useCallback 等方式来明确告知 React 组件在什么情况下需要重新渲染，尽量减少不必要的渲染。Fiber 架构也是为了解决这个问题。

Vue 在渲染过程中自动完成依赖绑定，取决于他的模板语法，同时维护状态和视图的依赖关系也会带来性能上的消耗。React 直接扩展了 JSX 来实现视图，结合函数式组件，又带来了更好的自由度。

所以从直观上，Vue 看起来更加自动化，写起来也更加容易，框架本身替你做了绝大多数工作。而 React 通常伴随着一些心智上的负担，虽然通过 hooks 和 api 提供了很多能力，但新手往往不清楚到底怎么做才是最好的。
