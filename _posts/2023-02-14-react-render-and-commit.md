---
title: "渲染和提交"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-02-14
last_modified_at: 2023-02-14
categories:
  - React
tags:
  - React
---

将 React 组件呈现到页面上需要经历下面三个步骤：

1. 触发渲染
2. 渲染组件
3. 提交到 DOM

## 触发渲染

以下两种情况会触发渲染：

1. 初次渲染

2. 组件（或者组件的祖先）状态发生改变

```jsx
import { createRoot } from "react-dom/client";
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1); // 状态改变触发再次渲染
  }

  return <button onClick={handleClick}>You pressed me {count} times</button>;
}

const root = createRoot(document.getElementById("root"));
root.render(<Counter />); // 初次渲染
```

### 安排渲染任务

触发渲染后，React 并没有立即开始渲染工作，而是将渲染任务做了计划，具体何时执行需要听从调度。

我们来看一看 React 大致是如何处理的。

_我们并非深入探究每行代码，只需理解 React 的工作原理_。

触发渲染后总会进入`scheduleUpdateOnFiber`函数：

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L697" target="_blank">跳转到 Github 查看源码</a>

```javascript
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
  eventTime: number
) {
  // ...省略

  ensureRootIsScheduled(root, eventTime);

  // ...省略
}
```

然后进入`ensureRootIsScheduled`：

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L865" target="_blank">跳转到 Github 查看源码</a>

```javascript
// Use this function to schedule a task for a root.
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // ...省略

  // Determine the next lanes to work on, and their priority.
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );

  // ...省略

  // We use the highest priority lane to represent the priority of the callback.
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  // ...省略

  // 判断优先级
  if (includesSyncLane(newCallbackPriority)) {
    // 优先级高
    // Special case: Sync React callbacks are scheduled on a special internal queue

    // ...省略

    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    if (supportsMicrotasks) {
      // 如果支持微任务
      // Flush the queue in a microtask.
      // ...省略
    } else {
      // 调用scheduler package的API以高优先级安排任务
      scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks);
    }
  } else {
    // 优先级不高
    // 调用scheduler package的API以其他优先级安排任务

    // ...省略确定优先级的代码

    scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
}
```

我们来看看这个函数的重要内容：

- 确定 Lanes 和优先级，_Lanes 模型是 React 内部调度的一个重要模型，暂时不深入了解_
  - 优先级很高：安排为同步回调，回调函数是`performSyncWorkOnRoot`
    - 如果支持微任务，会在微任务中来调用回调函数
    - 否则使用 scheduler 的 API 来实现回调
  - 优先级不高：安排为异步回调，回调函数是`performConcurrentWorkOnRoot`
    - 使用 scheduler 的 API 来实现回调

> 安排微任务可能按照次序使用`queueMicrotask`，`Promise`, `timeout`中的一种，取决于平台是否支持。<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-dom-bindings/src/client/ReactDOMHostConfig.js#L433" target="_blank">跳转到 Github 查看源码</a>

**scheduler**

`scheduler`是 React 中一个用于任务调度的包，现在仅在 React 中使用，但是完全可以独立出来作为通用调用算法。

> <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler" target="_blank">跳转到 Github 查看源码</a>

`scheduler`将任务放在最小堆中，每次拿出优先级最高的任务执行。

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerMinHeap.js" target="_blank">最小堆实现</a>

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerPriorities.js" target="_blank">定义的优先级</a>

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L346" target="_blank">查看添加回调任务的源码</a>

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L616" target="_blank">使用 MessageChannel 来调用回调</a>

_选择 MessageChannel 以为回调作为事件循环模型中的 task 执行，而不是 microtask_

## 渲染阶段

两种情况下 React 在渲染阶段做的工作：

- 初次渲染：创建 DOM 节点
- 再次渲染：计算与上一次渲染之间的差异，_此阶段不会使用差异信息做实际性修改，那是下一个阶段的工作_

在上面一个章节我们看到 React 为渲染安排了回调：`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`，它们的主要调用流程如下：

- `performSyncWorkOnRoot(root: FiberRoot)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1477" target="_blank">_call_</a>

  - `renderRootSync(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2076" target="_blank">_call_</a>
    - `workLoopSync()` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2121" target="_blank">_call_</a>
      - `performUnitOfWork(unitOfWork: Fiber)`

- `performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1055" target="_blank">_call_</a>

  - `renderRootConcurrent(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2251" target="_blank">_call_</a>
    - `workLoopConcurrent()` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2299" target="_blank">_call_</a>
      - `performUnitOfWork(unitOfWork: Fiber)`

无论是哪一种情况，最终都会进入`performUnitOfWork`函数，我们先不讨论这个函数，我们先看这个函数的上一层：工作循环。

### 工作循环

下面是这两种工作循环的代码。

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

**两种工作循环的差别**

它们的唯一区别是在调用`performUnitOfWork`前是否判断`shouldYield()`以便让出主线程，这个 API 是 scheduler package 提供的，它的精简代码如下：

```javascript
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    // The main thread has only been blocked for a really short amount of time;
    // smaller than a single frame. Don't yield yet.
    return false;
  }
  return true;
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L487" target="_blank">跳转到 Github 查看源码</a>

_贴士：源码中本来还有更多判断，例如使用 Facebook 和 Chrome 合作的 API `navigator.scheduling.isInputPending()`，但是由于 React 目前默认<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerFeatureFlags.js#L11" target="_blank">没有开启</a>这个功能，所以代码精简了。_

如果占用主线程时间超出`frameInterval`，那么就需要让出主线程，`frameInterval`的初始值是<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerFeatureFlags.js#L14" target="_blank">5ms</a>。

### performUnitOfWork 的参数

在分析 Fiber 前，我们先了解一些其他概念：

**声明式 vs 指令式**

React 使用声明式编写 UI（代替指令式），这使得开发者的工作变得更加容易，开发者只需要告诉 React 你需要显示出来的 UI 是什么样的，剩下的工作由 React 来完成。

在 React 的<a href="https://beta.reactjs.org/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative" target="_blank">文档</a>中举了一个例子来类比：你坐上一辆车需要到某个目的地，指令式就是告诉司机什么时候直行，什么时候转向，而声明式就是直接告诉司机目的地，他会自动把你送到目的地。

<figure>
  <figcaption>指令式</figcaption>
  <img src="/assets/images/i_imperative-ui-programming.png">
</figure>

<figure>
  <figcaption>声明式</figcaption>
  <img src="/assets/images/i_declarative-ui-programming.png">
</figure>

**Virtual DOM**

开发者只是声明了 UI，React 会创建一些”虚拟的“内容来描述 UI 并保存在内存中，元素和 Fiber 被认为是 Virtual DOM 实现的一部分。在 React 的<a href="https://zh-hans.reactjs.org/docs/faq-internals.html#what-is-the-virtual-dom" target="_blank">文档</a>由更多关于 Virtual DOM 的描述。

_贴士：其实叫 Virtul DOM 并不十分贴切，因为 React 并不是只能渲染为 DOM，还可以在移动平台上渲染为 native 视图_

**协调 reconciliation**

UI 从 A 状态变成 B 状态，React 需要计算出哪部分需要变化，而不是简单的重新渲染（提高性能），这个过程叫做协调。

_贴士：虽然不同的 Renderer 渲染出的内容差别很大，但是协调的算法应该尽可能相似。_

<figure>
  <figcaption>reconciler & renderer</figcaption>
  <img src="/assets/images/react_reconciler_renderer.jpg">
</figure>

在 React16 **以前**，React 使用的协调解决方案叫 Stack reconciler，这不是一个官方名称，只是由于他的处理方式与 Stack 很相似。

Stack reconciler 通过传入的元素创建了一些实例，然后再创建 DOM，更新的时候更新实例，然后更新 DOM。它的处理方式一层一层向下的，先处理本元素，然后处理子元素，或者处理调用函数组件或者类组件的 render 方法返回新的元素。React<a href="https://zh-hans.reactjs.org/docs/implementation-notes.html" target="_blank">文档</a>中有 Stack reconciler 的简单实现，有兴趣可以阅读。

<figure>
  <figcaption>Stack reconciler</figcaption>
  <img src="/assets/images/stack_reconciler.png">
</figure>

<figure>
  <img src="/assets/images/react_element_instance_dom.jpg">
</figure>

Stack reconciler 有很明显的局限性：

- 同步无法中断，如果中断了，这时浏览器需要重新绘制，那么可能导致 UI 不一致，例如上图中处理完第二个 item 后中断，这时候第一个 item 和第二个 item 的 DOM 已经发生变化了。

为了解决这些问题，React16 引入了**Fiber**，我们先看一看 Fiber reconciler 是如何做的，然后再去研究细节。

<figure>
  <figcaption>Fiber reconciler</figcaption>
  <img src="/assets/images/fiber_reconciler1.png">
  <img src="/assets/images/fiber_reconciler2.png">
</figure>

新的协调算法将可中断的任务切分成更小的任务，在执行完一个任务片段后可以让出主线程，如果有更高优先级的任务可以在这时候执行。

而且 Fiber reconciler 分成渲染和提交两个阶段，渲染阶段将所有的内容处理完成，然后在进入提交阶段进行统一渲染，这样就保证 UI 的一致性。渲染阶段可中断去执行其他任务，但是在提交阶段无法中断，否则无法保证 UI 的一致性。

<figure>
  <figcaption>React Phases</figcaption>
  <img src="/assets/images/react_phases.png">
</figure>

**Fiber 是什么**

Fiber 就是一个个小的任务单元（unit of work），你可以把它理解成原来 Stack 中的一帧，它受 React 控制：

- 可以暂停（pause），然后回来继续执行未完成的任务
- 可以中止（abort），注意与暂停的区别，这里指放弃掉
- 可以赋予优先级
- 可以重用

_贴士：更多 Fiber 的目标可以在 React<a href="https://zh-hans.reactjs.org/docs/codebase-overview.html#fiber-reconciler" target="_blank">文档</a>中查看_

**Fiber 的数据结构**

Fiber 是一个有以下属性的对象：

_贴士：这里仅仅列出一些重要属性，完整结构请查看<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactInternalTypes.js#L79" target="_blank">源码</a>_

- tag

  定义了 Fiber 的类型，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactWorkTags.js" target="_blank">源码</a>中定义了很多种类型，例如下面这几个常见类型:

  ```javascript
  export const FunctionComponent = 0;
  export const ClassComponent = 1;
  export const IndeterminateComponent = 2; // Before we know whether it is function or class
  export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
  export const HostComponent = 5;
  ```

- key and type

  key 和 type 在创建 Fiber 时都是从元素直接<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiber.js#L650" target="_blank">复制</a>过来的，`fiber.key = element.key; fiber.type = element.type`。

  函数组件和类组件的`type`就是他们自己，宿主组件的`type`是`string`，例如`div`, `span`。

- stateNode

  维护了 Fiber 的本地状态，例如 DOM 节点或者类组件的实例等等。

- return, child and sibling

  这三个属性使得不同的 Fiber 之间建立起了联系，`fiber.child`指向第一个子节点，`fiber.return`指向父节点，`fiber.sibling`指向下一个兄弟节点。

  <figure>
    <figcaption>Fiber tree</figcaption>
    <img src="/assets/images/fiber_structure.png">
  </figure>

- pendingProps and memoizedProps

  `pendingProps`是执行 Fiber 前设置的属性，`memoizedProps`是执行 Fiber 后设置的属性。如果二者相同，那么就表示上一次 Fiber 的输出可以重用，避免重复工作。

- alternate

  一个组件可能不止有一个 Fiber：`current` fiber 表示当前状态，`workInProgress` fiber 表示正在处理。`current.alternate === workInProgress`并且`workInProgress.alternate === current`。

**双缓冲**

React 在内部维护了两个版本的 Fiber Tree：

- current：已处理完成的版本
- workInProgress：正在处理的版本，下文以`wip`代替

它们之间使用`alternate`属性关联，这样做可以重用 Fiber 对象减少内存分配。

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiber.js#L266" target="_blank">源码</a>中创建`workInProgress`就体现了 React 使用了双缓冲解决方案。

### 源码分析 3：执行任务单元 peformUnitOfWork

上面我们说到 work loop 会调用<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2303" target="_blank">`performUnitOfWork(unitOfWork: Fiber)`</a>，第一个`unitOfWork`是`root`中保存的类型为`HostRoot`的 Fiber，下文都简称`HostRoot`。

下面是这个函数的精简版本：

```javascript
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  const next = beginWork(current, unitOfWork, ...);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    wip = next;
  }
}
```

这个函数非常好理解：

- 调用`beginWork`

  - 如果没有返回下一个 Fiber，那么就调用`completeUnitOfWork`

  - 否则让`wip`指向下一个 Fiber，进入下一次 work loop。

### 源码分析 4: beginWork

下面是这个函数的精简版本。

```typescript
function beginWork(
  current: Fiber | null,
  wip: Fiber,
  renderLanes: Lanes
): Fiber | null {
  // ...

  if (curren !== null) {
    // ...

    if (matchSomeConditon) {
      // 满足某些条件可以提前返回
      return attemptEarlyBailoutIfNoScheduledUpdate(current, wip, renderLanes);
    }
  } else {
    // ...
  }

  // ...

  switch (wip.tag) {
    // ...
    case FunctionComponent: {
      // ...
      return updateFunctionComponent(/*...*/);
    }
    case ClassComponent: {
      // ...
      return updateClassComponent(/*...*/);
    }
    case HostRoot: {
      // ...
      return updateHostRoot(/*...*/);
    }
    case HostComponent: {
      // ...
      return updateHostComponent(/*...*/);
    }
  }
}
```

**提前退出 beginWork**

工作循环始终从`HostRoot`开始，但是`wip`自身可能没有任何更新，这时也存在不同情况：

- 子树没有更新，返回`null`，然后进入`completeUnitOfWork`
- 子树有更新，返回`child`

_贴士：判断子树有没有更新可以通过判断<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L3609" target="_blank">workInProgress.childLanes</a> 属性，我们下面再说 lane 模型_
