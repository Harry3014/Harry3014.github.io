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

### 源码分析 2：工作循环

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

我们初次遇到了 Fiber 这个类型，稍后我们再详细说明。

### 源码分析 3：执行工作循环中的任务
