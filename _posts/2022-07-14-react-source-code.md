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

说明：本文分析基于<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a" target="_blank">React 18 版本</a>。

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

Element 是 React 中很重要的概念，它是构成 React 应用的最小单元，它可以是 host component（例如 html 的 div，span），可以是 function component，class component，fragment 等等内容。

我们可以通过 JSX，`React.createElement`来创建 element。说到 JSX，我们可以多了解一些关于它的内容。

我们都知道 JSX 依靠 Babel 等工具转换为 JavaScript。在 React17 以前，JSX 会被转换为`React.createElement`，这也是为什么必须要引入

上面的 JSX 创建了 React 元素，React 元素的大致结构如下，在<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react/src/ReactElement.js#L148" target="_blank">源码</a>中可以看到它的完整结构。

```javascript
reactElement = {
  $$typeof,
  type,
  key,
  ref,
  props,
};
```

**$$typeof**

\$\$typeof 是 Symbol 类型，用于标识这个对象是 React Element。

**type**

type 表示 React 元素的类型，类组件和函数组件的 type 就是它们自己本身，原生 HTML 标签的 type 是字符串，例如"div", "span"。

## 渲染 React 元素

无论是第一次渲染，还是更新组件，React 的工作都需要经历两个阶段：

1. 渲染阶段：这一阶段的工作主要由协调器完成，主要是计算出需要变化的内容。第一次渲染可以理解为与空白内容进行比较。
2. 提交阶段：渲染器根据渲染阶段的输出去调整 DOM。

<figure>
<img src="/assets/images/element_instance_dom.png" />
</figure>

## stack 协调器

在介绍新版本的协调器之前，我们先来了解一下 React16 之前使用的 stack 协调器，stack 协调器的处理方式是一个递归的过程，这样有一个很大的缺点，那就是不能中断此过程来让主线程来处理其他更高优先级的任务。如果这个过程的时间比较长，那么可能会引起掉帧等问题。下面这张图可以很形象的描述 stack 协调器的处理方式。

<figure>
<img src="/assets/images/stack_reconciler.png" />
</figure>

## fiber 协调器

fiber 协调器解决了 stack 协调器的问题，它可以把任务切片，在处理任务的过程中可以中断去执行其他高优先级的任务。

<figure>
<img src="/assets/images/fiber_reconciler1.png" />
</figure>

<figure>
<img src="/assets/images/fiber_reconciler2.png" />
</figure>

fiber 协调器主要依赖 fiber 架构，fiber 架构中每一个 React 元素都有一个对应的 FiberNode，一个 FiberNode 就代表了一个小的任务单元。

在调用<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-dom/src/client/ReactDOMRoot.js#L93" target="_blank">root.render</a>后，React 并没有立即执行第一个任务单元，而是把这个任务交给调度器去安排。

### FiberNode 结构

现在我们来看看 FiberNode 的主要结构，在<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactFiber.old.js#L119" target="_blank">源码</a>中可以查看完整的结构。

```javascript
const fiberNode = {
  tag: null,
  type: null,
  key: null,
  stateNode: null,

  child: null,
  sibling: null,
  return: null,

  alternate: null,
};
```

**tag**

tag 给 fiber 定义了一个标签，React 定义了很多种标签，每一种标签都有不同的处理方式。函数组件的`tag=0`，类组件的`tag=1`，在<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactWorkTags.js#L10" target="_blank">源码</a>中查看更多标签。

**type**

type 是 fiber 的类型，函数组件和类组件的 type 就是它们本身，原生组件例如 div 就是字符串`"div"`。

**child, sibing, return**

上面我们说了每一个 React 元素都有一个对应的 fiber，那么这三个属性可以把所有的 fiber 节点都串联起来。`child`指向第一个子节点，`sibing` 指向下一个兄弟节点，`return` 指向父节点。

<figure>
<img src="/assets/images/fiber_structure.png" />
</figure>

## 工作循环

在渲染阶段执行任务是以一个循环方式进行的，有同步的工作循环和异步并发式的工作循环，它们大致的代码如下。

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

`root._internalRoot.current`就保存了一个 fiber，这个 fiber 就是当前的 fiber root，现在我们要创建新的 fiber 树了，那我们就用这个 fiber 创建一个新的 fiber 作为`workInProgress`，<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1548" target="_blank">源码</a>。

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

这个函数的执行主要包含 3 个阶段。

1. beginWork。
2. completeUnitOfWork。
3. completeWork，_这个函数是在 completeWork 中调用的_。

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

## 渲染/协调

无论是第一次渲染还是后续修改 props/state 等引发的更新，都会经历两个阶段：

1. 渲染/协调阶段：这一个阶段构建了新的 fiber 树，并且对比当前的 fiber 树找出不同，这一个阶段是可以中断去做其他重要的任务。
2. 提交阶段：根据上一个阶段得出的对比结果，更新 DOM，这一个阶段不能中断。
   $$
