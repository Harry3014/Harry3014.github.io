---
title: "react优先级系统"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-03-18
last_modified_at: 2023-03-18
categories:
  - React
tags:
  - React
---

这边文章我们来分析 react 的优先级系统。react 中存在着多个优先级相关的内容，下面我们一一进行说明。

## Lanes 模型

### 数据结构

lane 最初是在 fiber reconciler 中提出的，是为了替换原先的 expiration time 模型，他的值是一个 32 位的二进制。

react 中定义了很多种 lane 优先级，数值越小优先级越高（0 是个例外）。

```javascript
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000010;
export const DefaultLane: Lane = /*                     */ 0b0000000000000000000000000100000;
```

<a href="https://github.com/facebook/react/pull/18796" target="_blank">Initial Lanes implementation by acdlite · Pull Request #18796 · facebook/react</a>

由于 lane 的值是二进制数值，所以通过位运算操作起来非常方便，下面列举几种代表操作。

```javascript
export function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;
}

export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane): boolean {
  return (a & b) !== NoLanes;
}

export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane): boolean {
  return (set & subset) === subset;
}

export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}

export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}

export function intersectLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a & b;
}
```

### 从触发渲染开始

**创建lane**

无论是初次渲染 updateContainer，还是调用了 useState 的 set 函数，class 组件的`this.setState`，useReducer 的 dispatch 函数，都会调用一个叫 requestUpdateLane 的函数创建 lane。

```javascript
export function requestUpdateLane(fiber: Fiber): Lane {
  // Special cases
  const mode = fiber.mode;
  if ((mode & ConcurrentMode) === NoMode) {
    return (SyncLane: Lane);
  } else if (
    !deferRenderPhaseUpdateToNextBatch &&
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    // This is a render phase update. These are not officially supported. The
    // old behavior is to give this the same "thread" (lanes) as
    // whatever is currently rendering. So if you call `setState` on a component
    // that happens later in the same render, it will flush. Ideally, we want to
    // remove the special case and treat them as if they came from an
    // interleaved event. Regardless, this pattern is not officially supported.
    // This behavior is only a fallback. The flag only exists until we can roll
    // out the setState warning, since existing code might accidentally rely on
    // the current behavior.
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }

  const isTransition = requestCurrentTransition() !== NoTransition;
  if (isTransition) {
    // The algorithm for assigning an update to a lane should be stable for all
    // updates at the same priority within the same event. To do this, the
    // inputs to the algorithm must be the same.
    //
    // The trick we use is to cache the first of each of these inputs within an
    // event. Then reset the cached values once we can be sure the event is
    // over. Our heuristic for that is whenever we enter a concurrent work loop.
    if (currentEventTransitionLane === NoLane) {
      // All transitions within the same event are assigned the same lane.
      currentEventTransitionLane = claimNextTransitionLane();
    }
    return currentEventTransitionLane;
  }

  // Updates originating inside certain React methods, like flushSync, have
  // their priority set by tracking it with a context variable.
  //
  // The opaque type returned by the host config is internally a lane, so we can
  // use that directly.
  const updateLane: Lane = (getCurrentUpdatePriority(): any);
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // This update originated outside React. Ask the host environment for an
  // appropriate priority, based on the type of event.
  //
  // The opaque type returned by the host config is internally a lane, so we can
  // use that directly.
  const eventLane: Lane = (getCurrentEventPriority(): any);
  return eventLane;
}
```

requestUpdateLane 函数中我们重点关注三类情况：

- 过渡（transition）

  返回的是 TransitionLane，他的优先级比所有的同步优先级都低

  ```javascript
  export const SyncHydrationLane: Lane = /*               */ 0b0000000000000000000000000000001;
  export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000010;

  export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000100;
  export const InputContinuousLane: Lane = /*             */ 0b0000000000000000000000000001000;

  export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000010000;
  export const DefaultLane: Lane = /*                     */ 0b0000000000000000000000000100000;

  export const SyncUpdateLanes: Lane = /*                 */ 0b0000000000000000000000000101010;

  const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000001000000;
  const TransitionLanes: Lanes = /*                       */ 0b0000000011111111111111110000000;
  const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000010000000;
  const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000100000000;
  const TransitionLane3: Lane = /*                        */ 0b0000000000000000000001000000000;
  const TransitionLane4: Lane = /*                        */ 0b0000000000000000000010000000000;
  const TransitionLane5: Lane = /*                        */ 0b0000000000000000000100000000000;
  const TransitionLane6: Lane = /*                        */ 0b0000000000000000001000000000000;
  const TransitionLane7: Lane = /*                        */ 0b0000000000000000010000000000000;
  const TransitionLane8: Lane = /*                        */ 0b0000000000000000100000000000000;
  const TransitionLane9: Lane = /*                        */ 0b0000000000000001000000000000000;
  const TransitionLane10: Lane = /*                       */ 0b0000000000000010000000000000000;
  const TransitionLane11: Lane = /*                       */ 0b0000000000000100000000000000000;
  const TransitionLane12: Lane = /*                       */ 0b0000000000001000000000000000000;
  const TransitionLane13: Lane = /*                       */ 0b0000000000010000000000000000000;
  const TransitionLane14: Lane = /*                       */ 0b0000000000100000000000000000000;
  const TransitionLane15: Lane = /*                       */ 0b0000000001000000000000000000000;
  const TransitionLane16: Lane = /*                       */ 0b0000000010000000000000000000000;
  ```

- 起源于 react 内部方法设置的优先级，例如 flushSync

  ```javascript
  function flushSync(fn) {
    const previousPriority = getCurrentUpdatePriority();
    try {
      // DiscreteEventPriority是SyncLane
      setCurrentUpdatePriority(DiscreteEventPriority);
      if (fn) {
        return fn();
      } else {
        return undefined;
      }
    } finally {
      setCurrentUpdatePriority(previousPriority);
    }
  }
  ```

- 触发渲染也可能由事件引起，我们在研究 react 的事件系统时提到过，react 将事件分成了不同的优先级，不同优先级的事件监听器不同。两类高优先级的事件触发后会设置当前优先级。

  ```javascript
  export opaque type EventPriority = Lane;

  export const DiscreteEventPriority: EventPriority = SyncLane;
  export const ContinuousEventPriority: EventPriority = InputContinuousLane;
  export const DefaultEventPriority: EventPriority = DefaultLane;
  export const IdleEventPriority: EventPriority = IdleLane;
  ```

  input，click 等事件优先级很高，scroll 等事件优先级次之，其他优先级较低。

  ```javascript
  export function getEventPriority(domEventName: DOMEventName): EventPriority {
    switch (domEventName) {
      // Used by SimpleEventPlugin:
      case "input":
      case "click":
        return DiscreteEventPriority;
      case "mousemove":
      case "scroll":
        return ContinuousEventPriority;
      default:
        return DefaultEventPriority;
    }
  }
  ```

  在事件触发后，调用绑定在 root container 上的事件监听器也可能会设置优先级。

  ```javascript
  function dispatchDiscreteEvent(
    domEventName: DOMEventName,
    eventSystemFlags: EventSystemFlags,
    container: EventTarget,
    nativeEvent: AnyNativeEvent
  ) {
    const previousPriority = getCurrentUpdatePriority();
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = null;
    try {
      setCurrentUpdatePriority(DiscreteEventPriority);
      dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
    } finally {
      setCurrentUpdatePriority(previousPriority);
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }

  function dispatchContinuousEvent(
    domEventName: DOMEventName,
    eventSystemFlags: EventSystemFlags,
    container: EventTarget,
    nativeEvent: AnyNativeEvent
  ) {
    const previousPriority = getCurrentUpdatePriority();
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = null;
    try {
      setCurrentUpdatePriority(ContinuousEventPriority);
      dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
    } finally {
      setCurrentUpdatePriority(previousPriority);
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }
  ```

- 起源于 react 外部，例如 host 事件，除了上面的两类高优先级事件，或者在高于root container的层级触发的事件。

  ```javascript
  export function getCurrentEventPriority(): EventPriority {
    const currentEvent = window.event;
    if (currentEvent === undefined) {
      return DefaultEventPriority;
    }
    return getEventPriority(currentEvent.type);
  }
  ```

**标记lane**

创建 update lane 后，会合并到 fiber.lanes 上，并且会向上遍历祖先合并到 childLanes。

```javascript
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  update: ConcurrentUpdate | null,
  lane: Lane
): void {
  // Update the source fiber's lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  // Walk the parent path to the root and update the child lanes.
  let parent = sourceFiber.return;
  let node = sourceFiber;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }

    node = parent;
    parent = parent.return;
  }

  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

在安排 fiber 更新任务时也会将 lane 合并到 FiberRoot.pendingLanes 上。

```javascript
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
  eventTime: number
) {
  // Mark that the root has a pending update.
  markRootUpdated(root, lane, eventTime);
}

export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number
) {
  root.pendingLanes |= updateLane;
}
```

**安排root任务时使用lane**

在给 root 安排一个 task 时，会获取 nextLanes，并且根据优先级绑定不同的回调函数。

- 如果包含同步优先级，绑定 performSyncWorkOnRoot
- 否则绑定 performConcurrentWorkOnRoot

```javascript
// Use this function to schedule a task for a root. There's only one task per
// root; if a task was already scheduled, we'll check to make sure the priority
// of the existing task is the same as the priority of the next level that the
// root has work on. This function is called on every update, and right before
// exiting a task.
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // Determine the next lanes to work on, and their priority.
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );

  // We use the highest priority lane to represent the priority of the callback.
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  if (includesSyncLane(newCallbackPriority)) {
    // 本质就是将回调函数放进一个队列中
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    if (supportsMicrotasks) {
      // 如果支持微任务，可以flushSyncCallbacks作为微任务
      // flushSyncCallbacks就是执行队列中的回调
      // Flush the queue in a microtask.
      scheduleMicrotask(() => {
        flushSyncCallbacks();
      });
    } else {
      // 如果不支持，那么调用scheduler的api安排一个超高优先级的宏任务
      // Flush the queue in an Immediate task.
      scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks);
    }
  } else {
    let schedulerPriorityLevel;
    // 这里会根据nextLanes的优先级获取scheduler对应的优先级，然后安排任务
    scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
}
```

**执行root任务时使用lane**

在真正处理 root 上的 task 时也一样获取 nextLanes，performConcurrent会使用lane判断是否应该切片，**目前只有transition和Suspense会切片**。

<a href="https://github.com/facebook/react/issues/24392#issuecomment-1237604765" target="_blank">transiton和Suspense切片</a>

```javascript
// This is the entry point for every concurrent task, i.e. anything that
// goes through Scheduler.
function performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean) {
  // Determine the next lanes to work on, using the fields stored
  // on the root.
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );
  if (lanes === NoLanes) {
    // Defensive coding. This is never expected to happen.
    return null;
  }

  // We disable time-slicing in some cases: if the work has been CPU-bound
  // for too long ("expired" work, to prevent starvation), or we're in
  // sync-updates-by-default mode.
  const shouldTimeSlice =
    !includesBlockingLane(root, lanes) &&
    !includesExpiredLane(root, lanes) &&
    !didTimeout;
  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes);
}
```

**渲染阶段使用lane**

渲染阶段主要是beginWork阶段使用lanes。

- 检查子树是否有待办任务，如果没有，可以返回 null 跳过

```javascript
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // Reuse previous dependencies
    workInProgress.dependencies = current.dependencies;
  }

  markSkippedUpdateLanes(workInProgress.lanes);

  // Check if the children have any pending work.
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // The children don't have any work either. We can skip them.
    return null;
  }

  // This fiber doesn't have work, but its subtree does. Clone the child
  // fibers and continue.

  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

- 更新状态时跳过优先级不够高的更新（包括 useState，useReducer，类组件状态更新）

```javascript
function updateReducer(reducer, initialArg, init) {
  let update = first;
  do {
    const updateLane = update.lane;
    const shouldSkipUpdate = !isSubsetOfLanes(renderLanes, updateLane);
    if (shouldSkipUpdate) {
      // Priority is insufficient. Skip this update.
    }
  }
}
```


