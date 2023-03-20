---
title: "深入类组件state"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-03-06
last_modified_at: 2023-03-06
categories:
  - React
tags:
  - React
---

之所以把类组件放在 hook 之后分析，是因为函数组件和 hook 目前是更普遍的做法。

## mount

在 beginWork 阶段，处理类组件 fiber 的函数为 updateClassComponent。

```javascript
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
) {
  const instance = workInProgress.stateNode;
  let shouldUpdate;
  if (instance === null) {
    // In the initial pass we might need to construct the instance.
    constructClassInstance(workInProgress, Component, nextProps);
    mountClassInstance(workInProgress, Component, nextProps, renderLanes);
    shouldUpdate = true;
  } else if (current === null) {
    // In a resume, we'll already have an instance we can reuse.
    shouldUpdate = resumeMountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderLanes
    );
  } else {
    shouldUpdate = updateClassInstance(
      current,
      workInProgress,
      Component,
      nextProps,
      renderLanes
    );
  }
  const nextUnitOfWork = finishClassComponent(
    current,
    workInProgress,
    Component,
    shouldUpdate,
    hasContext,
    renderLanes
  );
  return nextUnitOfWork;
}
```

参数 Component 就是类组件。

判断是否创建了类组件实例`workInProgress.stateNode === null`，如果没有的话首先需要创建类组件实例。

```javascript
function constructClassInstance(
  workInProgress: Fiber,
  ctor: any,
  props: any
): any {
  let instance = new ctor(props, context);
  workInProgress.memoizedState =
    instance.state !== null && instance.state !== undefined
      ? instance.state
      : null;
  adoptClassInstance(workInProgress, instance);
  return instance;
}
function adoptClassInstance(workInProgress: Fiber, instance: any): void {
  instance.updater = classComponentUpdater;
  workInProgress.stateNode = instance;
  // The instance needs access to the fiber so that it can schedule updates
  setInstance(instance, workInProgress);
}
```

在创建实例的过程中会将`instance.state`赋给`workInProgress.memoizedState`中保存。

在实例上设置 updater 也很重要`instance.updater = classComponentUpdater`，这关系到 state 的更新。

## 触发 update

类组件通过调用`setState`更新状态继而触发再次渲染，顺带说一句，调用`forceUpdate`强制触发渲染。

```javascript
Component.prototype.setState = function (partialState, callback) {
  if (
    typeof partialState !== "object" &&
    typeof partialState !== "function" &&
    partialState != null
  ) {
    throw new Error(
      "setState(...): takes an object of state variables to update or a " +
        "function which returns an object of state variables."
    );
  }

  this.updater.enqueueSetState(this, partialState, callback, "setState");
};

Component.prototype.forceUpdate = function (callback) {
  this.updater.enqueueForceUpdate(this, callback, "forceUpdate");
};
```

这两者都是调用 updater 的方法。

```javascript
const classComponentUpdater = {
  enqueueSetState(inst: any, payload: any, callback) {
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const lane = requestUpdateLane(fiber);

    const update = createUpdate(eventTime, lane);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }

    const root = enqueueUpdate(fiber, update, lane);
    if (root !== null) {
      scheduleUpdateOnFiber(root, fiber, lane, eventTime);
      entangleTransitions(root, fiber, lane);
    }

    if (enableSchedulingProfiler) {
      markStateUpdateScheduled(fiber, lane);
    }
  },
  enqueueForceUpdate(inst: any, callback) {
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
    const lane = requestUpdateLane(fiber);

    const update = createUpdate(eventTime, lane);
    update.tag = ForceUpdate;

    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }

    const root = enqueueUpdate(fiber, update, lane);
    if (root !== null) {
      scheduleUpdateOnFiber(root, fiber, lane, eventTime);
      entangleTransitions(root, fiber, lane);
    }
  },
};
```

他们调用的方法的内容几乎是相同的。

- 创建 update 对象
- 把 update 放进 fiber.updateQueue 中（确切的是 fiber.updateQueue.shared）
- 然后安排调度更新（scheduleUpdateOnFiber）

```javascript
function createUpdate(eventTime: number, lane: Lane): Update<mixed> {
  const update: Update<mixed> = {
    eventTime,
    lane,

    tag: UpdateState,
    payload: null,
    callback: null,

    next: null,
  };
  return update;
}

function enqueueUpdate<State>(
  fiber: Fiber,
  update: Update<State>,
  lane: Lane
): FiberRoot | null {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // Only occurs if the fiber has been unmounted.
    return null;
  }

  const sharedQueue: SharedQueue<State> = (updateQueue: any).shared;

  if (isUnsafeClassRenderPhaseUpdate(fiber)) {
    // This is an unsafe render phase update. Add directly to the update
    // queue so we can process it immediately during the current render.
    const pending = sharedQueue.pending;
    if (pending === null) {
      // This is the first update. Create a circular list.
      update.next = update;
    } else {
      update.next = pending.next;
      pending.next = update;
    }
    sharedQueue.pending = update;

    // Update the childLanes even though we're most likely already rendering
    // this fiber. This is for backwards compatibility in the case where you
    // update a different component during render phase than the one that is
    // currently renderings (a pattern that is accompanied by a warning).
    return unsafe_markUpdateLaneFromFiberToRoot(fiber, lane);
  } else {
    return enqueueConcurrentClassUpdate(fiber, sharedQueue, update, lane);
  }
}

function enqueueConcurrentClassUpdate(fiber, queue, update, lane) {
  var interleaved = queue.interleaved;

  if (interleaved === null) {
    // This is the first update. Create a circular list.
    update.next = update;
    // At the end of the current render, this queue's interleaved updates will
    // be transferred to the pending queue.

    pushConcurrentUpdateQueue(queue);
  } else {
    update.next = interleaved.next;
    interleaved.next = update;
  }

  queue.interleaved = update;
  return markUpdateLaneFromFiberToRoot(fiber, lane);
}
```

有没有发现这个内容其实与 useState hook 是一样的，dispatchSetState 也是创建一个 update 对象（结构相比这里略微不同）放进 hook 的 queue 中。

## update

```javascript
function updateClassInstance(
  current: Fiber,
  workInProgress: Fiber,
  ctor: any,
  newProps: any,
  renderLanes: Lanes
): boolean {
  const instance = workInProgress.stateNode;

  cloneUpdateQueue(current, workInProgress);

  const unresolvedOldProps = workInProgress.memoizedProps;
  const oldProps =
    workInProgress.type === workInProgress.elementType
      ? unresolvedOldProps
      : resolveDefaultProps(workInProgress.type, unresolvedOldProps);
  instance.props = oldProps;
  const unresolvedNewProps = workInProgress.pendingProps;

  const oldState = workInProgress.memoizedState;
  let newState = (instance.state = oldState);
  processUpdateQueue(workInProgress, newProps, instance, renderLanes);
  newState = workInProgress.memoizedState;

  if (typeof getDerivedStateFromProps === "function") {
    applyDerivedStateFromProps(
      workInProgress,
      ctor,
      getDerivedStateFromProps,
      newProps
    );
    newState = workInProgress.memoizedState;
  }

  const shouldUpdate =
    checkHasForceUpdateAfterProcessing() ||
    checkShouldComponentUpdate(
      workInProgress,
      ctor,
      oldProps,
      newProps,
      oldState,
      newState,
      nextContext
    );

  // Update the existing instance's state, props, and context pointers even
  // if shouldComponentUpdate returns false.
  instance.props = newProps;
  instance.state = newState;
  instance.context = nextContext;

  return shouldUpdate;
}
```

- 处理 updateQueue 获得 newState，处理的过程与 hook 的 queue 类似
- 如果需要，调用 get DerivedStateFromProps
- checkShouldComponentUpdate，如果需要，调用 shouldComponentUpdate
- 更新`instance.state`

最后进入 finishClassComponent，如果没有更新，那么可能提前退出 beginWork。如果有更新，调用 render 方法返回新的 children，然后就可以进入 reconcileChildren 了。
