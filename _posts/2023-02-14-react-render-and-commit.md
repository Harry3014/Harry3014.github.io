---
title: "渲染和提交 Render and Commit"
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

## 第一步：触发渲染

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

## 第二步：渲染阶段 Render Phase

浏览器渲染需要构建 DOM 树，那么：

- 初次渲染：`root.render(<Counter />)`提供给 React 的是元素，在渲染阶段 React 需要创建对应的 DOM 节点。
- 再次渲染：`setCount(count + 1)`状态发生改变，在渲染阶段 React 需要计算出哪些内容发生了变化。

无论是初次渲染，还是状态更新后触发的再次渲染，都不会立即执行真正的渲染，React 可能将其安排为事件循环中的一个任务（task），也可能安排为一个微任务（microtask），这部分内容将在其他文章中详细说明。

### 源码分析 1：渲染阶段的开始

渲染阶段的工作始于`function performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean)`或者`function performSyncWorkOnRoot(root: FiberRoot)`，这取决于渲染任务是异步还是同步。

_贴士：上一个段落中提及的任务（task）就是异步任务，微任务（microtask）是同步任务_

- `performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1055" target="_blank">_call_</a>
- `performSyncWorkOnRoot(root: FiberRoot)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1477" target="_blank">_call_</a>
  - `renderRootConcurrent(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2251" target="_blank">_call_</a>
  - `renderRootSync(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2076" target="_blank">_call_</a>
    - `workLoopConcurrent()` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2299" target="_blank">_call_</a>
    - `workLoopSync()` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2121" target="_blank">_call_</a>
      - <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2303" target="_blank">`performUnitOfWork(unitOfWork: Fiber)`</a>

### 源码分析 2：工作循环 work loop

无论是同步还是异步，最终都会调用`performUnitOfWork(unitOfWork: Fiber)`，我们先看这个函数的上一层：工作循环 work loop。

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

`workLoopConcurrent`与`workLoopSync`的唯一区别是在调用`performUnitOfWork`前是否判断<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/forks/Scheduler.js#L487" target="_blank">`shouldYield()`</a>以便让出主线程，`shouldYiled`的代码大致如下。

```javascript
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    return false;
  }
  return true;
}
```

_贴士：源码中本来还有更多判断，例如使用 Facebook 和 Chrome 合作的 API `navigator.scheduling.isInputPending()`，但是由于 React 目前默认<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerFeatureFlags.js#L11" target="_blank">没有开启</a>这个功能，所以代码精简了。_

如果超出`frameInterval`，那么就需要让出主线程，`frameInterval`的初始值是<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/scheduler/src/SchedulerFeatureFlags.js#L14" target="_blank">5ms</a>。

我们看到进入`performUnitOfWork`函数还有另外一个条件`workInProgress !== null`，第一个`workInProgress`在执行`renderRootSync(root: FiberRoot, lanes: Lanes)`或者`renderRootConcurrent(root: FiberRoot, lanes: Lanes)`时创建。

- `renderRootSync(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2029" target="_blank">_call_</a>
- `renderRootConcurrent(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2152" target="_blank">_call_</a>
  - `prepareFreshStack(root: FiberRoot, lanes: Lanes)` <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1738" target="_blank">_call_</a>
    - <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiber.js#L267" target="_blank">`createWorkInProgress(current: Fiber, pendingProps: any)`</a>

我们来看看这几个函数中的两个重要参数:

- `root: FiberRoot`：还记得我们通过`const root = createRoot(document.getElementById("root"))`创建的`root`对象吗？它的结构如下，`root._internalRoot`就是这里的`root`参数。

  <figure>
    <img src="/assets/images/react_dom_root_object.jpg">
  </figure>

- `current: Fiber`：`root.current`就是这个参数。_这里的 root 指的是上面的 root 参数。_

### Fiber

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

### 源码分析 3：执行任务单元 peformUnitOfWork

上面我们说到 work loop 会调用<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2303" target="_blank">`performUnitOfWork(unitOfWork: Fiber)`</a>，第一个`unitOfWork`是`root`中保存的类型为`HostRoot`的 Fiber，下文都简称`HostRoot`。

我们现在来看看初次渲染是怎样建立整个 Fiber 结构的，以及更新状态后再次渲染是怎样处理的。

`performUnitOfWork(unitOfWork: Fiber)`可以精简为下面的代码：

```javascript
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  const next = beginWork(current, unitOfWork, ...);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

这个函数非常好理解，调用`beginWork`，

- 如果没有返回下一个 Fiber，那么就调用`completeUnitOfWork`

- 否则让`workInProgress`指向下一个 Fiber，进入下一次 work loop。

在<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L3936" target="_blank">`beginWork`</a>中会<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L4031" target="_blank">根据不同类型的 Fiber 做不同的处理</a>。

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  // ...

  if (curren !== null) {
    // ...

    if (matchSomeConditon) {
      // 满足某些条件可以提前跳出，因为渲染阶段都从HostRoot开始，但是可能某些节点没有任何需要修改的内容，这样能提高性能
      return attemptEarlyBailoutIfNoScheduledUpdate(
        current,
        workInProgress,
        renderLanes
      );
    }
  } else {
    // ...
  }

  // ...

  switch (workInProgress.tag) {
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
