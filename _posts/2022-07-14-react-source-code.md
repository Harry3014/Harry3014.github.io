---
title: "深入React"
excerpt: ""
toc: true
toc_sticky: true
date: 2022-07-14
last_modified_at: 2022-07-14
categories:
  - React
tags:
  - React
---

说明：本文分析基于<a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d" target="_blank">React 18 版本</a>，而且只讨论 React 运行浏览器这一个平台上的情况。

## React 源码概述

React 源码由多个 package 组成，我们下面来介绍几个核心的 package。

### react

<a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react" target="_blank">react</a>包含了 React 的所有顶层 API，比如：

- 组件：Fragment，Suspense，Profiler，StrictMode。

- Hooks：useState，useEffect 等等。

- API：Component，createContext，createElement 等等。

### renderer

renderer 负责将组件渲染出来，不同的 render 在不同的平台上可以将组件渲染成不同的内容，比如：

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-dom" target="_blank">react-dom</a>渲染成 DOM。

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-native-renderer" target="_blank">react-native-renderer</a>渲染成 Native 视图。

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-test-renderer" target="_blank">react-test-renderer</a>渲染成 JSON 树。

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-art" target="_blank">react-art</a>渲染成矢量图。

### reconciler

<a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler" target="_blank">reconciler</a>做的工作：对比当前内容计算出变化的部分，然后将结果给 renderer 进行渲染，这个过程叫协调。

### scheduler

<a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler" target="_blank">scheduler</a>实现了协作式调度，可以先执行优先级高的任务，例如用户交互，将优先级低的任务延后执行，例如渲染刚从网络上下载的新内容。

## React 基础

我们先了解一些基础概念。

### 元素 Element

元素可以理解为 UI 的轻量化描述，可以通过 JSX 或者`React.createElement`来创建元素。

元素 object 包含下面一些属性：

- \$\$typeof：它的<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/shared/ReactSymbols.js#L15" target="_blank">值</a>是 symbol 类型，用于标识这个 object 是 React 元素。

- type: 它的值是`string | function | class`，分别对应原生组件，函数组件和类组件。

- props：保存了元素的一些属性。

- key：给 React 元素添加一个唯一标识可以帮助 React 区分同一个列表中的元素，这对于添加子项，删除子项，给子项排序是非常重要的。

元素更完整的结构见<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react/src/ReactElement.js#L148" target="_blank">源码</a>。

**非常重要！！**应该把元素当成不可变的对象，只要一经创建，内容不可修改。

举个例子，`<h1 className="greeting">Hello from <i>React</i></h1>`会创建一个元素：

```javascript
{
  // 省略$$typeof属性
  type: "h1",
  props: {
    className: "greeting",
    children: [
      "Hello from ",
      {
        $$typeof: Symbol(react.element),
        type: "i",
        props: {
          children: "React"
        }
        key: null,
        ref: null,
      }
    ]
  },
  key: null,
  ref: null
}
```

### JSX

JSX 依靠 Babel 等工具转换为 JavaScript。在 React17 以前，JSX 会被转换为`React.createElement`，这也是为什么必须要引入 react package 的原因。

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

## 两个重要阶段

以下两种情况会触发渲染：

- 初次渲染
- 组件（或者它的祖先）状态发生改变

无论是哪一种，React 做的工作可以分为两个阶段：

1. 渲染阶段
2. 提交阶段

下面我们来详细讨论这两个阶段。

## 渲染阶段

- 初次渲染期间，React 创建 DOM 节点。
- 后续重新渲染，React 计算出变化的内容，React 并不会用计算出的结果做出相应的改动。

下面我们来看 React 具体是如何实现的。

### 初次渲染入口

我们调用`react-dom/client的`<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-dom/src/client/ReactDOMRoot.js#L181" target="_blank">`creatRoot`</a>创建一个 root，然后调用它的 render 方法，例如：

```jsx
import { createRoot } from "react-dom/client";

function Greeting() {
  return <h1>Hello world</h1>;
}

const root = createRoot(document.getElementById("root"));
root.render(<Greeting />);
```

### 后续重新渲染入口

状态更新会导致重新渲染，例如：

```jsx
import { useState } from "react";
import { createRoot } from "react-dom/client";

function Button() {
  const [text, setText] = useState("off");

  function handleClick() {
    setText(text === "off" ? "on" : "off");
  }

  return <button onClick={handleClick}>{text}</button>;
}

const root = createRoot(document.getElementById("root"));
root.render(<Button />);
```

在点击按钮后调用`setText`函数会触发重新渲染。

**非常重要！！！**渲染的过程应该是纯净没有任何副作用的（否则会导致不可预测的行为），那么意味着：

1. 相同的输入要有相同的输出
2. 不应该修改渲染前的任何变量

下面我们深入了解渲染的过程。

### 第三步：提交修改到 DOM

提交阶段会修改 DOM，初次渲染会使用`appendChild()`，重新渲染时只修改最小变化的部分，这个信息在前面的渲染阶段已经计算好了。在修改 DOM 后浏览器就可以重新进行绘制了，然后就能呈现在屏幕上了。

下面我们介绍重要的两个阶段，渲染阶段和提交阶段。

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

那么 fiber 到底是什么呢？fiber 就是任务单元 unit of work，就是模拟原来 stack 协调器调用栈中一个一个 stack frame，但是 fiber 可以由 React 来调度什么时候暂停或者中止任务。

### fiber 的数据结构

下面是 fiber 的大致数据结构，在<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactInternalTypes.js#L79" target="_blank">源码</a>中可以查看完整的结构。

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

tag 表示 fiber 的类型，它决定了这个 fiber 应该怎样处理，React 定义了很多种类型，比如`FunctionComponent, ClassComponent, HostComponent`。在<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactWorkTags.js#L10" target="_blank">源码</a>中查看更多标签。

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

它们唯一的区别是：在执行任务前是否调用<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L487" target="_blank">shouldYield</a>，shouldYield 会检查主线程占据的时间是否已经超过了阈值，如果超过则交出主线程控制权方便执行其他高优先级的任务，例如浏览器绘制或者用户输入等等。

**workInProgress**

`workInProgress`表示正在处理的任务，我们看到需要满足`workInProgress !== null`条件才能执行任务，那么第一次渲染时第一个任务是什么呢？还记得我们调用<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-dom/src/client/ReactDOMRoot.js#L181" target="_blank">ReactDOM.createRoot</a>创建的 root 对象吗？

`root._internalRoot.current`就保存了一个 fiber，这个 fiber 叫 HostRoot，现在我们要创建新的 fiber 树了，那我们就用这个 fiber 创建一个新的 fiber 作为`workInProgress`，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1738" target="_blank">源码</a>。

**fiber 处理流程概览**

下面这四个函数概括了处理一个 fiber 的流程。

- <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2303" target="_blank">performUnitOfWork</a>
- <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L3936" target="_blank">beginWork</a>
- <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2472" target="_blank">completeUnitOfWork</a>
- <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCompleteWork.js#L872" target="_blank">completeWork</a>

**performUnitOfWork**

下面的代码是精简过的，但是也可以表示此函数的核心内容。

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

调用`beginWork`后如果有返回值，那么下一个任务就是这个返回的 fiber，普通情况下返回 child fiber，如果没有返回值则调用 `completeUnitOfWork`，我们先不详细的说明`beginWork`做了什么内容，先来看看`completeUnitOfWork`做了什么。

**completeUnitOfWork**

与`performUnitOfWork`一样，下面的代码是精简过的。

```javascript
function completeUnitOfWork(unitOfWork) {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  let completedWork = unitOfWork;
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // Check if the work completed or if something threw.
    if (completedCondition) {
      const next = completeWork(current, completedWork);

      if (next !== null) {
        // Completing this fiber spawned new work. Work on that next.
        workInProgress = next;
        return;
      }
    } else {
      // This fiber did not complete because something threw.
      const next = unwind(current, completedWork);

      if (next !== null) {
        workInProgress = next;
        return;
      }
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

代码中的注释都是 react 源码中的原文，"Attempt to complete the current unit of work, then move to the next sibling. If there are no more siblings, return to the parent fiber."。这一句注释已经能够说明`completeUnitOfWork`，尝试完成当前的这个任务，如果有兄弟节点，则下一个要处理的任务就是兄弟节点，否则返回父 fiber。

**示例**

假设有下面这样一颗 fiber 树。

<figure>
<img src="/assets/images/performUnitOfWork.png" />
</figure>

那么这四个函数的执行过程应该如下。

```javascript
workInProgress = HostRoot;
performUnitOfWork(HostRoot);
beginWork(HostRoot);

next = App;
workInProgress = App;
performUnitOfWork(App);
beginWork(App);

next = Header;
workInProgress = Header;
performUnitOfWork(Header);
beginWork(Header);

next = null;
completedWork = Header;
completeUnitOfWork(Header);
completeWork(Header);

next = Content;
workInProgress = Content;
performUnitOfWork(Content);
beginWork(Content);

next = null;
completedWork = Content;
completeUnitOfWork(Content);
completeWork(Content);

next = null;
completedWork = App;
completeWork(App);

next = null;
completedWork = HostRoot;
completeWork(HostRoot);
```

`beginWork`和`completeWork`才是执行任务单元的核心部分。

## React 任务调度

调度器接收到一个<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L346" target="_blank">安排请求</a>时，会根据提供的优先级创建一个新的任务，然后将这个任务放进任务队列中等待后续的安排。

React 有<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L87" target="_blank">两个任务队列</a>`taskQueue`和`timeQueue`，`timeQueue`是用于存放延时任务，这两个任务队列都是用<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerMinHeap.js" target="_blank">最小堆</a>实现的优先队列。

### 任务优先级

上面的任务队列为什么要使用优先队列呢？因为每个任务都有优先级，需要让优先级高的任务先执行。

React 的任务分为 5 个优先级，由高到低依次是：

1. ImmediatePriority 立即执行
2. UserBlockingPriority 用户阻塞
3. NormalPriority 普通优先级
4. LowPriority 低优先级
5. IdlePriority 空闲

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerPriorities.js#L10" target="_blank">源码</a>中声明了 6 个优先级，但是<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L27" target="_blank">实际</a>只使用了 5 个。

### 进入工作循环

把任务放进任务队列后，如果满足执行任务的<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L424" target="_blank">条件</a>，那么就可以准备执行任务了。

在浏览器环境中如果支持`MessageChannel`，那么可以就可以通过`MessagePort.postMessage`发送一条消息，然后`MessagePort.onMessage`在收到消息后进入工作循环，如果不支持，那么就通过`setTimeout`安排一个 0 秒后的定时任务。<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L618" target="_blank">源码</a>中说明了为什么优先使用`MessageChannel`，因为在<a href="https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers" target="_blank">HTML5 标准</a>中注明了在嵌套了 5 次定时任务后，至少会有 4ms 的延时。

进入工作循环后从任务队列中取出优先级最高的任务准备执行，为什么说准备执行呢？因为真正执行任务是要满足一些<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L217" target="_blank">条件</a>的，如果执行任务已经超过了一个限制时间，那么应该把主线程的控制权让给更优先的任务，例如浏览器绘制，用户输入等等。
