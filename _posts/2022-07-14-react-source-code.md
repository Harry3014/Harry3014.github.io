---
title: "深入React"
excerpt: ""
toc: true
date: 2022-07-14
last_modified_at: 2022-07-14
categories:
  - React
tags:
  - React
---

说明：本文分析基于<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a" target="_blank">React 18 版本</a>，而且只讨论 React 运行浏览器这一个平台上的情况。

## 总览 React 源码

React 源码由多个 package 组成，我们下面来介绍几个核心的 package。

### react

<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react" target="_blank">react</a>定义了 React 的顶层 API，例如`React.Component, React.createElement, React.Suspense`以及 Hook 等等。

### renderer

不同的 render 可以将 React 组件渲染成不同的内容，它们运行在不同的平台上。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-dom" target="_blank">react-dom</a>渲染成 DOM。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-native-renderer" target="_blank">react-native-renderer</a>渲染成 Native 视图。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-test-renderer" target="_blank">react-test-renderer</a>渲染成 JSON 树。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-art" target="_blank">react-art</a>渲染成矢量图。

### reconciler

在第一次渲染或者需要更新时，<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler" target="_blank">react-reconciler</a>对比当前内容找出变化的部分，然后提供给 renderer 高效地更新渲染内容。

到了这里你可能有一个疑问，第一次渲染跟谁比较呢？答案是跟空白内容进行比较。

### scheduler

<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler" target="_blank">scheduler</a>按照不同的优先级合理的安排任务的执行，还可以中止任务把主线程交给优先级更高的任务。

## React 基础

在进入更深的探讨之前，我们有必要了解一些 React 的基础但是很核心的概念。

### React Element

元素是 React 中很重要的概念，它是构成 React 应用的最小单元，它可以是 host component（例如 html 的 div，span），可以是 function component，class component，fragment 等等内容。

我们可以通过 JSX，`React.createElement`来创建元素。说到 JSX，我们可以多了解一些关于它的内容。

我们都知道 JSX 依靠 Babel 等工具转换为 JavaScript。在 React17 以前，JSX 会被转换为`React.createElement`，这也是为什么必须要引入 react package 的原因。

假设源代码如下：

```jsx
import React from "react";

function App() {
  return <h1>Hello world</h1>;
}
```

旧的 JSX 转换会转换成下面的结果：

```javascript
import React from "React";

function App() {
  return React.createElement("h1", null, "Hello world");
}
```

新的 JSX 可以不用引入 react package，会转换为下面的结果：

```jsx
function App() {
  return <h1>Hello world</h1>;
}
```

```javascript
// 由编译器自动引入，不允许手动引入
import { jsx as _jsx } from "react/jsx-runtime";

function App() {
  return _jsx("h1", { children: "Hello world" });
}
```

上面的 JSX 创建了 React 元素，React 元素的大致结构如下，在<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react/src/ReactElement.js#L148" target="_blank">源码</a>中可以看到它的完整结构。

```
{
  $$typeof,
  type,
  key,
  ref,
  props,
}
```

**$$typeof**

\$\$typeof 的值是`Symbol(react.element)`，用于标识这个 object 是 React 元素。

**type**

type 是 React 元素的类型。

**key**

给 React 元素添加一个唯一标识可以帮助 React 区分同一个列表中的元素，这对于添加子项，删除子项，给子项排序是非常重要的。

**props**

`props`中保存了元素的一些属性。

下面我列出了一些常见类型组件会创建出什么样的 React 元素。

```html
<h1>Hello world from <i>React</i></h1>
```

```javascript
{
  $$typeof: Symbol(react.element),
  type: "h1",
  key: null,
  ref: null,
  props: {
    children: [
      "Hello World from ",
      {
        $$typeof: Symbol(react.element),
        type: "i",
        key: null,
        ref: null,
        props: {
          children: "React"
        }
      }
    ]
  }
}
```

```jsx
function App() {
  return <h1>Hello world</h1>;
}

class App extends React.Component {
  render() {
    return <h1>Hello world</h1>;
  }
}
```

```javascript
{
  $$typeof: Symbol(react.element),
  type: App,
  key: null,
  ref: null,
  props: {}
}
```

## 渲染组件的三个步骤

把组件渲染到屏幕上显示会经历下面三个步骤：

1. 触发渲染
2. 渲染组件
3. 提交到 DOM

<figure>
<img src="/assets/images/render_commit.png" />
</figure>

我们先简单说明一下每个阶段会做哪些事情，后面我们会针对渲染和提交阶段做详细的分析。

### 触发渲染

有两种方式可以触发渲染：

1. 初次渲染
2. 组件状态改变后重新渲染

### 渲染组件

渲染组件的过程就是调用组件的过程，初次渲染是从根组件开始，后续的渲染从状态改变的组件开始。调用的过程是递归的，组件返回了子组件，那么会继续渲染子组件。

在初次渲染时会创建 DOM 节点，在重新渲染时 React 会计算出与上一次渲染的差别，这一步我们不会使用比较信息做任何事情，那是下一个阶段的工作。

渲染的过程可能会被 React 暂停或者重新启动，我们会在详细分析中解释 React 为什么这样做。

**非常重要！！！**渲染的过程应该是纯净没有任何副作用的，那么意味着：

1. 相同的输入要有相同的输出
2. 不应该修改渲染前的任何变量

### 提交修改到 DOM

提交阶段会修改 DOM，初次渲染会使用`appendChild()`，重新渲染时只修改最小变化的部分，这个信息在前面的渲染阶段已经计算好了。在修改 DOM 后浏览器就可以重新进行绘制了，然后就能呈现在屏幕上了。

## 渲染阶段

渲染阶段的核心工作由协调器完成，在 React16 以前使用的是 stack 协调器。

### stack 协调器

我们不探究这个旧的协调器是如何实现的，下面这张图可以很形象的描述它的工作机制，它有一些缺点：它是同步的不能中断，不能将任务切片，假设一次性需要处理的任务过多，可能会导致掉帧。

<figure>
<img src="/assets/images/stack_reconciler.png" />
</figure>

### fiber 协调器

我们来看看比较理想的情况是怎样的。

- 能够把任务切片
- 能够暂停任务，也能够在后续恢复任务的执行
- 能够区分任务的优先级，让优先级高的任务先执行

下面的图就描述了 fiber 协调器的工作原理，在执行了一个切片后的任务后，可以交出主线程的控制权去执行其他优先级高的任务。

<figure>
<img src="/assets/images/fiber_reconciler1.png" />
</figure>

<figure>
<img src="/assets/images/fiber_reconciler2.png" />
</figure>

那么 fiber 到底是什么呢？fiber 就是切片任务的抽象结构，就是模拟原来 stack 协调器调用栈中一个一个 stack frame，但是 fiber 可以由 React 来调度什么时候暂停或者中止任务。

### fiber 的数据结构

下面是 fiber 的大致数据结构，在<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactFiber.old.js#L119" target="_blank">源码</a>中可以查看完整的结构。

```javascript
{
  tag,
  key,
  type,
  stateNode,

  child,
  sibling,
  return,

  alternate,
}
```

**tag**

tag 表示 fiber 的类型，它决定了这个 fiber 应该怎样处理，React 定义了很多种类型，比如`FunctionComponent, ClassComponent, HostComponent`。在<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactWorkTags.js#L10" target="_blank">源码</a>中查看更多标签。

**key, type**

key 和 type 都是直接从 React 元素中复制过去的。

**stateNode**

如果是类组件，那么 stateNode 就是创建的实例，如果是 host component，那么就是创建的 DOM 节点。

**child, sibing, return**

这三个属性可以将 fiber 串联起来形成 fiber tree。`child`指向第一个子节点，`sibing` 指向下一个兄弟节点，`return` 指向父节点。

<figure>
<img src="/assets/images/fiber_structure.png" />
</figure>

**alternate**

fiber 架构采用了 double buffering，一个元素在内存中可能存在两个对应的 fiber，一个是当前显示内容对应的 current，一个是即将需要处理的 workInProgress，用伪代码来表示即：`current.alternate === workInProgress`，`workInProgress.alternate === current`。

### 工作循环

在第一次渲染时我们要给每个元素创建对应的 fiber，在后续发生更新时我们要找出变化的部分。

无论是第几次渲染，React 都使用通用的处理方式——工作循环，有同步的工作循环和异步并发式的工作循环，它们大致的代码如下。

```javascript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

**workInProgress**

`workInProgress`表示正在处理的任务，我们看到需要满足`workInProgress !== null`条件才能执行任务，那么第一次渲染时第一个任务是什么呢？还记得我们调用<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-dom/src/client/ReactDOMRoot.js#L167" target="_blank">ReactDOM.createRoot</a>创建的 root 对象吗？

`root._internalRoot.current`就保存了一个 fiber，这个 fiber 叫 HostRoot，现在我们要创建新的 fiber 树了，那我们就用这个 fiber 创建一个新的 fiber 作为`workInProgress`，<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1548" target="_blank">源码</a>。

**performUnitOfWork**

这个函数的大致结构如下，<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1907" target="_blank">源码</a>

```javascript
function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;

  let next = beginWork(current, unitOfWork);

  if (next === null) {
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

我们来看渲染下面这个示例时 workLoop 是怎么运作的。

```html
<h1>
  <span>hello</span>
  <i>world</i>
</h1>
```

<figure>
<img src="/assets/images/reconciler_workloop.png" />
</figure>

**beginWork**

## React 任务调度

调度器接收到一个<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L308" target="_blank">安排请求</a>时，会根据提供的优先级创建一个新的任务，然后将这个任务放进任务队列中等待后续的安排。

React 有<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L72" target="_blank">两个任务队列</a>`taskQueue`和`timeQueue`，`timeQueue`是用于存放延时任务，这两个任务队列都是用<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/SchedulerMinHeap.js" target="_blank">最小堆</a>实现的优先队列。

### 任务优先级

上面的任务队列为什么要使用优先队列呢？因为每个任务都有优先级，需要让优先级高的任务先执行。

React 的任务分为 5 个优先级，由高到低依次是：

1. ImmediatePriority 立即执行
2. UserBlockingPriority 用户阻塞
3. NormalPriority 普通优先级
4. LowPriority 低优先级
5. IdlePriority 空闲

<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/SchedulerPriorities.js" target="_blank">源码</a>中声明了 6 个优先级，但是<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L24" target="_blank">实际</a>只使用了 5 个。

### 进入工作循环

把任务放进任务队列后，如果满足执行任务的<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L381" target="_blank">条件</a>，那么就可以准备执行任务了。

在浏览器环境中如果支持`MessageChannel`，那么可以就可以通过`MessagePort.postMessage`发送一条消息，然后`MessagePort.onMessage`在收到消息后进入工作循环，如果不支持，那么就通过`setTimeout`安排一个 0 秒后的定时任务。<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L569">源码</a>中说明了为什么优先使用`MessageChannel`，因为在<a href="https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers" target="_blank">HTML5 标准</a>中注明了在嵌套了 5 次定时任务后，至少会有 4ms 的延时。

进入工作循环后从任务队列中取出优先级最高的任务准备执行，为什么说准备执行呢？因为真正执行任务是要满足一些<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L197" target="_blank">条件</a>的，如果执行任务已经超过了一个限制时间，那么应该把主线程的控制权让给更优先的任务，例如浏览器绘制，用户输入等等。
