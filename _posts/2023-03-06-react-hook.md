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
function beginWork(current: Fiber | null, workInProgress: Fiber, renderLanes: Lanes) {
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component ? unresolvedProps : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(current, workInProgress, Component, resolvedProps, renderLanes);
    }
  }
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L3936" target="_blank">查看完整源码</a>

updateFunctionComponent 调用 renderWithHooks 返回 children，然后进入 children 的协调。

```javascript
function updateFunctionComponent(
  current: null | Fiber,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
) {
  let nextChildren = renderWithHooks(current, workInProgress, Component, nextProps, context, renderLanes);

  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L1081" target="_blank">查看完整源码</a>

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
    current === null || current.memoizedState === null ? HooksDispatcherOnMount : HooksDispatcherOnUpdate;
  let children = Component(props, secondArg);
  return children;
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L419" target="_blank">查看完整源码</a>

**useState**

我们以分析最常见的 useState 为例。

```javascript
function useState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>] {
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

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L2790" target="_blank">多种 dispatcher</a>

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
  const dispatch = (queue.dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue));
  return [hook.memoizedState, dispatch];
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L1815" target="_blank">查看完整源码</a>

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

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L819" target="_blank">查看完整源码</a>

workInProgressHook 和 currentHook 变量分别保存了两个 hook list。

```javascript
// Hooks are stored as a linked list on the fiber's memoizedState field. The
// current hook list is the list that belongs to the current fiber. The
// work-in-progress hook list is a new list that will be added to the
// work-in-progress fiber.
let currentHook: Hook | null = null;
let workInProgressHook: Hook | null = null;
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L227" target="_blank">查看完整源码</a>

初始化 hook 后，返回带有 initialState 和 set 函数的数组。

**触发 update**

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

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L2605" target="_blank">查看完整源码</a>

**fiber & queue**

fiber 和 queue 都是 mountState 时绑定在 dispatchSetState 函数上的。

**action**

action 是 nextState，可能是一个函数。

set 函数的核心内容是：

- 创建一个 udpate 对象，update 对象的结构如下，next 指向下一个 update 形成一个链表

  ```javascript
  type Update<S, A> = {
    lane: Lane,
    action: A,
    hasEagerState: boolean,
    eagerState: S | null,
    next: Update<S, A>,
  };
  ```

- 把 update 放进 hook 的 queue 中

_本来还有一种情况可以提前跳出，这里省略了_

无论如何，最终 queue 的结构如下：

<figure>
  <img src="/assets/images/hook-queue.png" />
</figure>
如果不是渲染阶段调用更新，那么就安排新的渲染。

**update**

进入渲染阶段后，调用函数组件再次调用 useState，这时的 dispatcher 调用 updateState 函数。

```javascript
function updateState(initialState) {
  return updateReducer(basicStateReducer);
}
```

可能你会好奇，为什么 updateState 调用的是 updateReducer？

答案：因为他们的目的是相同的，都是返回一个新的 state，useState 的状态更新逻辑也可以写为一个 reducer 函数，区别仅仅是 state reducer 的 action 参数并非标准的带有 type 属性的对象。

```javascript
function basicStateReducer(state, action) {
  return typeof action === "function" ? action(state) : action;
}
```

现在我们来看看 updateReducer 函数。

```javascript
function updateReducer(reducer, initialArg, init) {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  if (queue === null) {
    throw new Error("Should have a queue. This is likely a bug in React. Please file an issue.");
  }

  queue.lastRenderedReducer = reducer;

  const current = currentHook;

  // The last rebase update that is NOT part of the base state.
  let baseQueue = current.baseQueue;

  // The last pending update that hasn't been processed yet.
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    // We have new updates that haven't been processed yet.
    // We'll add them to the base queue.
    if (baseQueue !== null) {
      // Merge the pending queue and the base queue.
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  if (baseQueue !== null) {
    // We have a queue to process.
    const first = baseQueue.next;
    let newState = current.baseState;

    let update = first;
    do {
      // Process this update.
      const action = update.action;
      if (update.hasEagerState) {
        // If this update is a state update (not a reducer) and was processed eagerly,
        // we can use the eagerly computed state
        newState = update.eagerState;
      } else {
        newState = reducer(newState, action);
      }
      update = update.next;
    } while (update !== null && update !== first);

    // Mark that the fiber performed work, but only if the new state is
    // different from the current state.
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    hook.memoizedState = newState;
  }

  const dispatch = queue.dispatch;
  return [hook.memoizedState, dispatch];
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L1061" target="_blank">查看完整源码</a>

**fiber & queue**

- 获取当前的 hook
- hook.baseQueue 中可能还有之前的 update，一并把 pendingQueue 合并到 baseQueue 中
- 遍历 baseQueue，调用 reducer 得到 newState
- 比较 newState 和 hook.memoizedState，不同才表示有更新（这个标志会在 updateFunctionComponent 中使用到）
- 返回 newState 和 dispatch 函数

如果有兴趣了解，下面是获取当前 hook 的代码。

```javascript
function updateWorkInProgressHook() {
  // This function is used both for updates and for re-renders triggered by a
  // render phase update. It assumes there is either a current hook we can
  // clone, or a work-in-progress hook from a previous render pass that we can
  // use as a base. When we reach the end of the base list, we must switch to
  // the dispatcher used for mounts.
  let nextCurrentHook;

  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;

    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }

  let nextWorkInProgressHook;

  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // There's already a work-in-progress. Reuse it.
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;
    currentHook = nextCurrentHook;
  } else {
    // Clone from the current hook.
    if (nextCurrentHook === null) {
      throw new Error("Rendered more hooks than during the previous render.");
    }

    currentHook = nextCurrentHook;
    const newHook = {
      memoizedState: currentHook.memoizedState,
      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,
      next: null,
    };

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      currentlyRenderingFiber$1.memoizedState = workInProgressHook = newHook;
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }

  return workInProgressHook;
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L840" target="_blank">查看完整源码</a>

### useReducer

**mount**

```javascript
function mountReducer(reducer, initialArg, init) {
  const hook = mountWorkInProgressHook();
  let initialState;

  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = initialArg;
  }

  hook.memoizedState = hook.baseState = initialState;
  const queue = {
    pending: null,
    interleaved: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  const dispatch = (queue.dispatch = dispatchReducerAction.bind(null, currentlyRenderingFiber, queue));
  return [hook.memoizedState, dispatch];
}
```

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L1032" target="_blank">查看完整源码</a>

逻辑基本与 mountState 一致。

**触发 update**

与触发 update state 基本一致，只是没有提前跳出的部分。

```javascript
function dispatchReducerAction(fiber, queue, action) {
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

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberHooks.js#L2566" target="_blank">查看完整源码</a>

**update**

useState 和 useReducer 共用 updateReducer。

### useEffect

**mount**

```javascript
function mountEffect(create: () => (() => void) | void, deps: Array<mixed> | void | null): void {
  mountEffectImpl(PassiveEffect | PassiveStaticEffect, HookPassive, create, deps);
}

function mountEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, undefined, nextDeps);
}

function pushEffect(
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | void | null
): Effect {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: (null: any),
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```

这个过程其实很简单

- 创建 effect 对象，他的结构如下，我们一一说明他的属性

  ```javascript
  type Effect = {
    tag: HookFlags,
    create: () => (() => void) | void,
    destroy: (() => void) | void,
    deps: Array<mixed> | void | null,
    next: Effect,
  };
  ```

  **tag**

  effect 的类型，因为不止 useEffect 这一种副作用，还有 layout effect，insertion effect 等等。

  ```javascript
  export const NoFlags = /*   */ 0b0000;

  // Represents whether effect should fire.
  export const HasEffect = /* */ 0b0001;

  // Represents the phase in which the effect (not the clean-up) fires.
  export const Insertion = /* */ 0b0010;
  export const Layout = /*    */ 0b0100;
  export const Passive = /*   */ 0b1000;
  ```

  **create**

  setup 函数

  **destroy**

  cleanup 函数

  **deps**

  依赖

  **next**

  下一个 effect

- 将 effect 放入`fiber.updateQueue`中，`fiber.updateQueue.lastEffect`指向最后一个 effect，`fiber.updateQueue.lastEffect.next`指向第一个 effect，所以他是一个循环链表。

**update**

```javascript
function updateEffect(create: () => (() => void) | void, deps: Array<mixed> | void | null): void {
  updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, destroy, nextDeps);
}
```

update 与 mount 类似，但是需要判断依赖是否改变，如果没有改变，那么此 effect 将不会添加 HookHasEffect 这个 flag。

**执行 effect**

我们在介绍渲染提交阶段提到过，渲染阶段需要保持纯净，副作用都在提交阶段执行。

在提交阶段有多个入口执行副作用，可能同步也可能异步（异步的可能性高），都是在提交阶段三个子阶段完成后才执行，执行副作用的函数一样的。

```javascript
function flushPassiveEffectsImpl() {
  commitPassiveUnmountEffects(root.current);
  commitPassiveMountEffects(root, root.current, lanes, transitions);
}
```

- 卸载副作用：执行 cleanup 函数
- 挂载副作用：执行 setup 函数，返回值作为 effect.destroy

执行的过程与 layout effect 一样是递归的。

### useContext

**创建 context**

```javascript
function createContext(defaultValue) {
  const context = {
    $$typeof: REACT_CONTEXT_TYPE,
    _currentValue: defaultValue,
    Provider: null,
    Consumer: null,
  };
  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };
  context.Consumer = context;
  return context;
}
```

创建 context 对象，他的属性应该都比较好理解。

**提供 context**

使用`<SomeContext.Provider value={newValue}>`提供 context。在 beginWork 中处理如下。

```javascript
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
  }
}

function updateContextProvider(current, workInProgress, renderLanes) {
  const context = providerType._context;

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;

  const newValue = newProps.value;

  // 更新context._currentValue = newValue
  context._currentValue = newValue;

  if (oldProps !== null) {
    const oldValue = oldProps.value;
    if (is(oldValue, newValue)) {
      // No change. Bailout early if children are the same.
      if (oldProps.children === newProps.children && !hasLegacyContextChanged()) {
        return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
      }
    } else {
      // The context value changed. Search for matching consumers and schedule
      // them to update.
      propagateContextChange(workInProgress, context, renderLanes);
    }
  }

  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

**使用 context**

返回 context.\_currentValue

## 范例

### useReducer

```js
const [state, dispatch] = useReducer(reducer, initialArg, init?)
```

- reducer 函数，接收当前 state 和一个 action，action 的结构可以多种多样，但是一般都含有 type 属性来描述发生了什么，根据不同 action 返回新的 state。

```js
function reducer(state, action) {
  // ...
}
```

- 如果没有 init 参数，那么初始 state 就是 initialArg，否则调用 init 返回初始 state

下面的例子使用了 immer。

```js
import { useImmerReducer } from "use-immer";
import AddTask from "./AddTask.js";
import TaskList from "./TaskList.js";

function tasksReducer(draft, action) {
  switch (action.type) {
    case "added": {
      draft.push({
        id: action.id,
        text: action.text,
        done: false,
      });
      break;
    }
    case "changed": {
      const index = draft.findIndex((t) => t.id === action.task.id);
      draft[index] = action.task;
      break;
    }
    case "deleted": {
      return draft.filter((t) => t.id !== action.id);
    }
    default: {
      throw Error("未知 action：" + action.type);
    }
  }
}

export default function TaskApp() {
  const [tasks, dispatch] = useImmerReducer(tasksReducer, initialTasks);

  function handleAddTask(text) {
    dispatch({
      type: "added",
      id: nextId++,
      text: text,
    });
  }

  function handleChangeTask(task) {
    dispatch({
      type: "changed",
      task: task,
    });
  }

  function handleDeleteTask(taskId) {
    dispatch({
      type: "deleted",
      id: taskId,
    });
  }

  return (
    <>
      <h1>布拉格的行程安排</h1>
      <AddTask onAddTask={handleAddTask} />
      <TaskList tasks={tasks} onChangeTask={handleChangeTask} onDeleteTask={handleDeleteTask} />
    </>
  );
}

let nextId = 3;
const initialTasks = [
  { id: 0, text: "参观卡夫卡博物馆", done: true },
  { id: 1, text: "看木偶戏", done: false },
  { id: 2, text: "打卡列侬墙", done: false },
];
```

### useContext

1. **创建** context
2. **提供** context
3. **使用** context

context 的使用场景：

- **主题**： 如果你的应用允许用户更改其外观（例如暗夜模式），你可以在应用顶层放一个 context provider，并在需要调整其外观的组件中使用该 context。
- **当前账户**： 许多组件可能需要知道当前登录的用户信息。将它放到 context 中可以方便地在树中的任何位置读取它。某些应用还允许你同时操作多个账户（例如，以不同用户的身份发表评论）。在这些情况下，将 UI 的一部分包裹到具有不同账户数据的 provider 中会很方便。
- **路由**： 大多数路由解决方案在其内部使用 context 来保存当前路由。这就是每个链接“知道”它是否处于活动状态的方式。如果你创建自己的路由库，你可能也会这么做。
- **状态管理**： 随着你的应用的增长，最终在靠近应用顶部的位置可能会有很多 state。许多遥远的下层组件可能想要修改它们。通常 将 reducer 与 context 搭配使用来管理复杂的状态并将其传递给深层的组件来避免过多的麻烦。

下面介绍 reducer 和 context 结合的例子，上面的 TaskApp 还需要传递 tasks 和事件处理程序。

现在我们修改一下，把 tasks 状态和 dispatch 函数放入 context 中，这样就无需透传了。

1. 创建 context

TaskContext.js

```jsx
import { createContext } from "react";

export const TasksContext = createContext(null);
export const TasksDispatchContext = createContext(null);
```

2. 提供 context

TaskApp.js

```jsx
import { useReducer } from "react";
import AddTask from "./AddTask.js";
import TaskList from "./TaskList.js";
import { TasksContext, TasksDispatchContext } from "./TasksContext.js";

export default function TaskApp() {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  return (
    <TasksContext.Provider value={tasks}>
      <TasksDispatchContext.Provider value={dispatch}>
        <h1>Day off in Kyoto</h1>
        <AddTask />
        <TaskList />
      </TasksDispatchContext.Provider>
    </TasksContext.Provider>
  );
}
```

3. 使用 context

AddTask.js

```jsx
export default function AddTask() {
  const dispatch = useContext(TasksDispatchContext);

  // ...
}
```

TaskList.js

```jsx
export default function TaskList() {
  const tasks = useContext(TasksContext);
  // ...
}
```

可以通过将所有传递信息的代码移动到单个文件中来进一步整理组件。

- 可以导出一个像 TasksProvider 提供 context 的组件。
- 可以导出像 useTasks 和 useTasksDispatch 这样的自定义 Hook。

```jsx
export function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  return (
    <TasksContext.Provider value={tasks}>
      <TasksDispatchContext.Provider value={dispatch}>{children}</TasksDispatchContext.Provider>
    </TasksContext.Provider>
  );
}

export function useTasks() {
  return useContext(TasksContext);
}

export function useTasksDispatch() {
  return useContext(TasksDispatchContext);
}
```
