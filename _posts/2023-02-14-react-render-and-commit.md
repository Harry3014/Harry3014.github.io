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

**为什么选择 MessageChannel**

为了实现 0ms 延时的定时器，如果选择`setTimeout(fn, 0)`，可能是无法做到的，更多可以了解<a href="https://developer.mozilla.org/zh-CN/docs/Web/API/setTimeout#%E5%AE%9E%E9%99%85%E5%BB%B6%E6%97%B6%E6%AF%94%E8%AE%BE%E5%AE%9A%E5%80%BC%E6%9B%B4%E4%B9%85%E7%9A%84%E5%8E%9F%E5%9B%A0%EF%BC%9A%E6%9C%80%E5%B0%8F%E5%BB%B6%E8%BF%9F%E6%97%B6%E9%97%B4" target="_blank">MDN</a>上的解析。

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

### work loop

下面是这两种工作循环的代码。

> 跳转到 Github 查看源码

```javascript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

> 跳转到 Github 查看源码

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

**workInProgess**

`workInProgress`就是当前需要处理的 Fiber，可以调用下面的函数创建它：

```javascript
// This is used to create an alternate fiber to do work on.
function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode
    );

    // ...省略代码
  } else {
    // ...省略代码
  }

  // ...省略代码

  return workInProgress;
}
```

我们从 React 源码中的注释中看到这个函数使用了双缓冲池，如果`current.alternate === null`时才创建新的 Fiber，否则可以重用之前创建的 Fiber。

那么进入工作循环的第一个`workInProgress`是什么呢？

答案是：`renderRootSync`或`renderRootConcurrent`中调用`prepareFreshStack`创建。

```javascript
function prepareFreshStack(root: FiberRoot, lanes: Lanes): Fiber {
  // ...省略代码
  const rootWorkInProgress = createWorkInProgress(root.current, null);
  workInProgress = rootWorkInProgress;
  // ...省略代码
}
```

这里`FiberRoot`类型的`root`参数是哪里来的呢？

答案是：调用‘react-dom’ package 的 Client API `createRoot(domNode, options?)`时创建的，`reactDOMRoot._internalRoot`就是`FiberRoot`，`FiberRoot.current`是 Fiber 节点，我们称它为 HostRoot。

<figure>
  <figcaption>FiberRoot</figcaption>
  <img src="/assets/images/react_dom_root_object.jpg">
</figure>

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

### performUnitOfWork

如果满足工作循环的判断，那么就会进入`performUnitOfWork`，下面是这个函数的精简版本：

```javascript
function performUnitOfWork(unitOfWork) {
  let next = beginWork(unitOfWork);
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

这个函数非常好理解：

- 调用`beginWork`

  - 如果没有返回下一个 Fiber，那么就调用`completeUnitOfWork`

  - 否则让`workInProgress`指向下一个 Fiber，进入下一次工作循环。

**beginWork**

我们暂时不深入了解这个函数，现在我们仅仅需要关注它的返回值，它始终返回`workInProgress.child`或者`null`，值得注意的是`workInProgress.child`也可能是`null`。

**completeUnitOfWork**

如果 beginWork 返回 null，意味着这个分支已经没有需要处理的 Fiber 了，那么就可以完成当前这个 Fiber，然后可以接着处理它的兄弟节点，然后返回父节点。

下面是这个函数的精简版本：

```javascript
function completeUnitOfWork(unitOfWork) {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  let completedWork = unitOfWork;
  do {
    const next = completeWork(unitOfWork);
    if (next !== null) {
      // Completing this fiber spawned new work. Work on that next.
      workInProgress = next;
      return;
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }

    // Otherwise, return to the parent
    completedWork = completedWork.return;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

### 渲染阶段结束

到这里渲染阶段就结束了，可以看到实际工作是在 beginWork 和 completeWork 中完成的，但是我们目前还没有深入了解这两个函数。

可能到这里我们有一个疑问，渲染阶段到底产出了什么呢？

答案是：生成了一个“全新”的 Fiber tree，之所以加引号，是因为并非所有的 Fiber 都是新创建的，可能是重用了之前的 Fiber，其中的 Fiber 有可能还标记了副作用。

`FiberRootNode.finishedWork`还指向了这个新的 Fiber tree，这在下一个阶段中有使用到。

**什么是副作用**

副作用这个词从字面上很难理解，<a href="https://beta.reactjs.org/learn/synchronizing-with-effects#what-are-effects-and-how-are-they-different-from-events" target="_blank">React 文档</a>和<a href="https://zh.wikipedia.org/zh-hans/%E5%89%AF%E4%BD%9C%E7%94%A8_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6)" target="_blank">维基百科</a>中有一些关于副作用的解释。在计算机科学中，副作用表示对于函数外的变量，修改了参数等等，例如在事件处理函数中更改状态，发送 http 请求，导航到其他页面等等都是副作用。我们熟知的 Hook 还有类组件的一些生命周期方法都是副作用。

还记得我们之前提到渲染阶段必须是纯函数，不能有任何副作用，否则 UI 将不受控制，所以在渲染阶段只是将副作用标记在 Fiber 上，等进入提交阶段再执行副作用。

Fiber 对象中有一些属性就是专门为副作用设置的：

```javascript
{
  // Effect
  flags: Flags,
  subtreeFlags: Flags,
  deletions: Array<Fiber> | null,

  // Singly linked list fast path to the next fiber with side-effects.
  nextEffect: Fiber | null,

  // The first and last fiber with side-effect within this subtree. This allows
  // us to reuse a slice of the linked list when we reuse the work done within
  // this fiber.
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,
}
```

`flags`中就保存了副作用的 flag，例如`Placement | Update | ChildDeletion`，值得注意的是`flags`中可能保存了很多副作用。

原来的设计中，属性`nexeEffect`使得有副作用的 Fiber 可以串联成一个链，但是后来不再使用`nextEffect | firstEffect | lastEffect`，而是去遍历整颗树。

## 提交阶段

提交阶段可以拆分成下面几个子阶段：

- before mutation 阶段

  对 host tree（例如 DOM 树）做出修改前，例如类组件的`getSnapshotBeforeUpdate`在这个阶段被调用。

- mutation 阶段

  插入，修改，删除 DOM 节点等等。

- layout 阶段

  修改 host tree 后，在浏览器进行绘制前，例如类组件的`componentDidMount | componentDidUpdate`在这个阶段被调用。

提交阶段主要包括在`commitRoot`函数中，它的精简版本如下：

```javascript
function commitRoot(root: FiberRoot) {
  const finishedWork = root.finishedWork;
  // The commit phase is broken into several sub-phases. We do a separate pass
  // of the effect list for each phase: all mutation effects come before all
  // layout effects, and so on.

  // The first phase a "before mutation" phase. We use this phase to read the
  // state of the host tree right before we mutate it. This is where
  // getSnapshotBeforeUpdate is called.
  commitBeforeMutationEffects(root, finishedWork);

  // The next phase is the mutation phase, where we mutate the host tree.
  commitMutationEffects(root, finishedWork);

  // The work-in-progress tree is now the current tree. This must come after
  // the mutation phase, so that the previous tree is still current during
  // componentWillUnmount, but before the layout phase, so that the finished
  // work is current during componentDidMount/Update.
  root.current = finishedWork;

  // The next phase is the layout phase, where we call effects that read
  // the host tree after it's been mutated. The idiomatic use case for this is
  // layout, but class component lifecycles also fire here for legacy reasons.
  commitLayoutEffects(finishedWork, root, lanes);
}
```
