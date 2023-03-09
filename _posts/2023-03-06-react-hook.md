---
title: "深入hook"
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

## 设计动机

## 数据结构

hook 保存在 fiber.memoizedState 中。

```javascript
export type Hook = {
  memoizedState: any,
  baseState: any,
  baseQueue: Update<any, any> | null,
  queue: any,
  next: Hook | null,
};
```

**next**

hook 被设计为一个链表，next 就指向了下一个 hook。

## 调用 hook api

**hook 使用规则**

- 只能在函数组件和自定义 hook 中调用 hook
- 只能在函数最顶层调用，不能在循环、条件判断或者子函数中调用

**函数组件**

我们以函数组件举例，函数组件在 beginWork 中调用 updateFunctionComponent。

```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes
      );
    }
  }
}
```

updateFunctionComponent 调用 renderWithHooks 返回 children，然后进入 children 的协调。

```javascript
function updateFunctionComponent(
  current: null | Fiber,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
) {
  let nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes
  );

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

renderWithHooks 会调用 Component 函数，在函数组件中可以调用 hook api。

```javascript
function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes
): any {
  currentlyRenderingFiber = workInProgress;
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  // Using memoizedState to differentiate between mount/update only works if at least one stateful hook is used.
  // Non-stateful hooks (e.g. context) don't get added to memoizedState,
  // so memoizedState would be null during updates and mounts.
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;
  let children = Component(props, secondArg);
  return children;
}
```

**useState**

我们以分析最常见的 useState 为例。

```javascript
function useState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

其他 hook 与 useState 相似之处是：在真正进入 hook api 代码前，需要先获取 dispatcher。

**hook dispatcher**

如果看的仔细，我们在 renderWithHooks 函数中就能发现在这里已经确定了 dispatcher，判断的条件是：`current === null || current.memoizedState === null`。

- 挂载时的 dispatcher 是 HooksDispatcherOnMount
- 更新时的 dispatcher 是 HooksDispatcherOnUpdate

```javascript
const HooksDispatcherOnMount: Dispatcher = {
  readContext,

  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useInsertionEffect: mountInsertionEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  useDebugValue: mountDebugValue,
  useDeferredValue: mountDeferredValue,
  useTransition: mountTransition,
  useMutableSource: mountMutableSource,
  useSyncExternalStore: mountSyncExternalStore,
  useId: mountId,
};
```

```javascript
const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,

  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useInsertionEffect: updateInsertionEffect,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  useDebugValue: updateDebugValue,
  useDeferredValue: updateDeferredValue,
  useTransition: updateTransition,
  useMutableSource: updateMutableSource,
  useSyncExternalStore: updateSyncExternalStore,
  useId: updateId,
};
```

他们对应的 hook api 的实际函数是不同的，例如`HooksDispatcherOnMount.useState = mountState`，然而`HooksDispatcherOnUpdate.useState = updateState`。_源码中还有其他的 dispatcher_

### useState

**mount**

```javascript
function mountState(initialState) {
  const hook = mountWorkInProgressHook();

  if (typeof initialState === "function") {
    initialState = initialState();
  }

  hook.memoizedState = hook.baseState = initialState;
  const queue = {
    pending: null,
    interleaved: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  const dispatch = (queue.dispatch = dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue
  ));
  return [hook.memoizedState, dispatch];
}
```

mount 时肯定需要先创建一个 hook 对象，不仅 mountState 这样处理，mountEffect 等等也是一样的。

```javascript
function mountWorkInProgressHook() {
  const hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}
```

workInProgressHook 和 currentHook 变量分别保存了两个 hook list。

```javascript
// Hooks are stored as a linked list on the fiber's memoizedState field. The
// current hook list is the list that belongs to the current fiber. The
// work-in-progress hook list is a new list that will be added to the
// work-in-progress fiber.
let currentHook: Hook | null = null;
let workInProgressHook: Hook | null = null;
```

初始化 hook 后，返回带有 initialState 和 set 函数的数组。

**update**

调用 set 函数触发 state 的更新，我们看看 set 函数。

```javascript
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);
  const update = {
    lane: lane,
    action: action,
    hasEagerState: false,
    eagerState: null,
    next: null,
  };

  if (isRenderPhaseUpdate(fiber)) {
    enqueueRenderPhaseUpdate(queue, update);
  } else {
    const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);
    if (root !== null) {
      const eventTime = requestEventTime();
      scheduleUpdateOnFiber(root, fiber, lane, eventTime);
      entangleTransitionUpdate(root, queue, lane);
    }
  }
}
```

**fiber & queue**

fiber 和 queue 都是 mountState 时绑定在 dispatchSetState 函数上的。

**action**

action 是 nextState，可能是一个函数。

set 函数的核心内容是：

- 创建一个 udpate 对象
- 把 update 放进 hook 的 queue 中

无论如何，最终 queue 的结构如下：

<figure>
  <img src="/assets/images/hook-queue.png" />
</figure>
如果不是渲染阶段调用更新，那么就安排新的渲染。

下面我们开始分析state的更新，更新state调用的是dispatcher的updateState函数。


```javascript
function updateState(initialState) {
  return updateReducer(basicStateReducer);
}
```

useState 和 useReducer 都是关于 state，useState 的状态更新逻辑也可以写为一个 reducer 函数，思考一下为什么？

答案：因为他们的目的是相同的，都是返回一个新的 state，区别仅仅是 state reducer 的 action 参数并非标准的带有 type 属性的对象。

```javascript
function basicStateReducer(state, action) {
  return typeof action === "function" ? action(state) : action;
}
```
