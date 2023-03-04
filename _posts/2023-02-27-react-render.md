---
title: "深入渲染阶段"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-02-27
last_modified_at: 2023-02-27
categories:
  - React
tags:
  - React
---

渲染阶段的主要实质性工作在 beginWork 和 completeWork 这两个函数完成。

_本文中的代码片段几乎都是精简版本，如需查看完整源码，点击代码片段下的链接跳转查看。_

## beginWork

```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (oldProps !== newProps || hasLegacyContextChanged()) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // Neither props nor legacy context changes. Check if there's a pending
      // update or context change.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );

      if (!hasScheduledUpdateOrContext) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes
        );
      }
    }
  } else {
    didReceiveUpdate = false;
  }

  switch (workInProgress.tag){
    ...
    case FunctionComponent:
        return updateFunctionComponent(...);
    case ClassComponent:
        return updateClassComponent(...);
    case HostRoot:
      return updateHostRoot(...);
    ...
  }
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L3936" target="_blank">查看完整源码</a>

### 参数说明

**current**

当前已渲染内容中的 fiber，可能是 null

**workInProgress**

此次渲染过程中待处理的 fiber

**renderLanes**

Lanes 模型，与优先级相关，_此文中不讨论这个内容_

### 提前退出 beginWork

条件 1：`current !== null`

条件 2：props 和 context 没有变化，_React 源码中的注释原文_

满足条件最终会调用 bailoutOnAlreadyFinishedWork。

```javascript
function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
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

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L3592" target="_blank">查看完整源码</a>

两种情况：

- 如果 workInProgress 的 children 没有待处理的工作，可以返回 null 继而调用 completeUnitWork。

- 否则，clone current 的 children（包括 sibling），返回 workInProgress.child。

思考一个问题：为什么要 clone current 的 children 呢？

答案：因为创建 workInProgress 时，指向的是 current.child，现在需要返回的是待处理的 fiber。

```javascript
function createWorkInProgress(current) {
  ...
  workInProgress.child = current.child;
  ...
  return workInProgress;
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiber.js#L324" target="_blank">查看完整源码</a>

### 根据 workInProgress.tag 做不同处理

如果不满足提前返回的条件，那么会根据 fiber 的类型做不同的处理，例如调用 updateHostRoot， updateFunctionComponent，updateClassComponent 等等。

**类组件**

类组件在此期间需要创建实例，并保存在 workInProgress.stateNode 中。

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L1275" target="_blank">查看完整源码</a>

多数常见类型的 fiber 会进入 reconcileChildren 函数，这就是 React 概念中的协调过程，其中就包含了如雷贯耳的 diffing 算法。

```javascript
function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes
    );
  }
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberBeginWork.js#L316" target="_blank">查看完整源码</a>

reconcileChildren 根据判断条件`current === null`分为两种情况处理，但他们的目的是一致的：构造 nextChildren 对应的 fiber。所以他们调用的函数实际都由 createChildReconciler 函数创建。

```javascript
function createChildReconciler(
  shouldTrackSideEffects: boolean
): ChildReconciler {
  function reconcileSingleElement() {}

  function reconcileChildrenArray() {}

  function reconcileSingleTextNode() {}

  function reconcileChildFibers() {}

  return reconcileChildFibers;
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactChildFiber.js#L267" target="_blank">查看完整源码</a>

### diffing 算法

reconcileChildren 实际就是 React 说的协调过程，这其中包含了如雷贯耳的 diffing 算法。

**设计动机**

在某一时间节点触发渲染，会创建一棵 fiber 树。在下一次触发渲染时，会产生一棵不同的 fiber 树。React 需要基于这两棵树之间的差别来判断如何高效的更新 UI，以保证当前 UI 与最新的树保持同步。

此算法有一些通用的解决方案，即生成将一棵树转换成另一棵树的最小操作次数。然而，即使使用最优的算法，该算法的复杂程度仍为 O(n 3 )，其中 n 是树中元素的数量。

如果在 React 中使用该算法，那么展示 1000 个元素则需要 10 亿次的比较。这个开销实在是太过高昂。于是 React 在以下两个假设的基础之上提出了一套 O(n) 的启发式算法：

- 开发者可以使用 key 属性标识哪些子元素在不同的渲染中可能是不变的
- 两个不同类型的元素会产生出不同的树

所以比较两个 fiber，核心就是比较 key 和 type。

下面这个函数是协调子节点的核心入口。

```javascript
// This API will tag the children with the side-effect of the reconciliation
// itself. They will be added to the side-effect list as we pass through the
// children and the parent.
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  // Handle object types
  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes
          )
        );
    }

    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }
  }

  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        "" + newChild,
        lanes
      )
    );
  }

  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactChildFiber.js#L1250" target="_blank">查看完整源码</a>

这个函数根据 newChild 的类型做不同的处理：

**object 类型**

有可能是 React 元素，调用 reconcileSingleElement

**数组**

调用 reconcileChildrenArray

**有迭代器**

调用 reconcileChildrenIterator，处理与数组类似

**字符串或数字**

调用 reconcileSingleTextNode

**不满足任何条件**

删除所有 children，fiber.deletions 数组中保存需要删除的 fiber，这个 fiber 指的是 parent。

### 单节点协调

**单个 React 元素的协调**

```javascript
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    if (child.key === key) {
      const elementType = element.type;
      if (child.elementType === elementType) {
        deleteRemainingChildren(returnFiber, child.sibling);
        const existing = useFiber(child, element.props);
        existing.ref = coerceRef(returnFiber, child, element);
        existing.return = returnFiber;
        return existing;
      }
      // Didn't match.
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.ref = coerceRef(returnFiber, currentFirstChild, element);
  created.return = returnFiber;
  return created;
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactChildFiber.js#L1137" target="_blank">查看完整源码</a>

遍历子节点，如果 key 和 type 都能匹配，返回此 child 的 alternate fiber（如果是 null 就创建新的 fiber）。

如果没有匹配，也创建新的 fiber。

在返回前，如果存在 sibling，需要删除，因为我们只有一个 React 元素，所以 fiber 树的这个分支也仅仅只可能有一个子节点。

**单个文本节点的协调**

```javascript
function reconcileSingleTextNode(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  textContent: string,
  lanes: Lanes
): Fiber {
  // There's no need to check for keys on text nodes since we don't have a
  // way to define them.
  if (currentFirstChild !== null && currentFirstChild.tag === HostText) {
    // We already have an existing node so let's just update it and delete
    // the rest.
    deleteRemainingChildren(returnFiber, currentFirstChild.sibling);
    const existing = useFiber(currentFirstChild, textContent);
    existing.return = returnFiber;
    return existing;
  }
  // The existing first child is not a text node so we need to create one
  // and delete the existing ones.
  deleteRemainingChildren(returnFiber, currentFirstChild);
  const created = createFiberFromText(textContent, returnFiber.mode, lanes);
  created.return = returnFiber;
  return created;
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactChildFiber.js#L1113" target="_blank">查看完整源码</a>

文本节点的协调不比较 key，因为没有地方给文本节点设置 key。

所以如果第一个 child 是文本类型，返回此 child 的 alternate fiber，否则创建新的 fiber，并且也要删除多余的 sibling。

其实还有其他类型的单节点协调，这里就不一一介绍了，可以查看<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactChildFiber.js#L1250" target="_blank">reconcileChildFibers</a>的完整源码。

### 多节点协调

数组类型或者有迭代器的 newChild 都属于多节点协调，下面以数组类型为例，迭代器处理与数组的处理从本质上来说一样的。

```javascript
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<any>,
  lanes: Lanes
): Fiber | null {
  let resultingFirstChild: Fiber | null = null;
  let previousNewFiber: Fiber | null = null;

  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    const newFiber = updateSlot(
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      lanes
    );
    if (newFiber === null) {
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }
    if (shouldTrackSideEffects) {
      if (oldFiber && newFiber.alternate === null) {
        // We matched the slot, but we didn't reuse the existing fiber, so we
        // need to delete the existing child.
        deleteChild(returnFiber, oldFiber);
      }
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  if (newIdx === newChildren.length) {
    // We've reached the end of the new children. We can delete the rest.
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }

  if (oldFiber === null) {
    // If we don't have any more existing children we can choose a fast path
    // since the rest will all be insertions.
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) {
        continue;
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }

  // Add all children to a key map for quick lookups.
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  // Keep scanning and use the map to restore deleted items as moves.
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes
    );
    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          // The new fiber is a work in progress, but if there exists a
          // current, that means that we reused the fiber. We need to delete
          // it from the child list so that we don't add it to the deletion
          // list.
          existingChildren.delete(
            newFiber.key === null ? newIdx : newFiber.key
          );
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  if (shouldTrackSideEffects) {
    // Any existing children that weren't consumed above were deleted. We need
    // to add them to the deletion list.
    existingChildren.forEach((child) => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactChildFiber.js#L744" target="_blank">查看完整源码</a>

**按顺序依次匹配**

取 newChildren[newIdx]与 oldFiber 调用 updateSlot 判断是否匹配。

```javascript
function updateSlot(
  returnFiber: Fiber,
  oldFiber: Fiber | null,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  // Update the fiber if the keys match, otherwise return null.

  const key = oldFiber !== null ? oldFiber.key : null;

  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    // Text nodes don't have keys. If the previous node is implicitly keyed
    // we can continue to replace it without aborting even if it is not a text
    // node.
    if (key !== null) {
      return null;
    }
    return updateTextNode(returnFiber, oldFiber, "" + newChild, lanes);
  }

  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE: {
        if (newChild.key === key) {
          return updateElement(returnFiber, oldFiber, newChild, lanes);
        } else {
          return null;
        }
      }
      ...
    }

    if (isArray(newChild) || getIteratorFn(newChild)) {
      if (key !== null) {
        return null;
      }

      return updateFragment(returnFiber, oldFiber, newChild, lanes, null);
    }
  }

  return null;
}
```

updateSlot 中还是靠判断 key 是否匹配来决定返回值。

- key 不匹配，返回 null，此次遍历结束
- key 匹配
  - type 匹配，复用 fiber
  - type 不匹配，新建 fiber

可以想象为两个数组分别由两个指针从前到后移动处理，key 不匹配就跳出。

**已到达 newChildren 终点**

表示已经遍历完 newChildren ，那么就可以直接返回第一个 childFiber。

**已到达 oldFiber 终点**

如果`oldFiber === null`，那么 newChildren 剩余的 child 创建新的 fiber。

**oldFiber 和 newChildren 都没有到终点**

剩余 oldFiber 放进 Map 中，索引是 key 或者 index。

遍历剩余的 newChidren：

- 使用 newChildren[newIdx].key 或 newIdx 去 Map 中获取 oldFiber
  - oldFiber 不存在，新建 fiber
  - oldFiber 存在，判断 type 是否匹配
    - 不匹配，新建 fiber
    - 匹配，复用 fiber

### begin 阶段的副作用标记

在 begin 阶段的协调，我们只涉及到 fiber 的创建、移动、删除，所以 fiber.flags 只涉及 Placement，ChildDeletion。

## completeWork

```javascript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  switch (workInProgress.tag) {
    case HostComponent: {
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(current, workInProgress, type, newProps);
      } else {
        const currentHostContext = getHostContext();
        const rootContainerInstance = getRootHostContainer();
        const instance = createInstance(
          type,
          newProps,
          rootContainerInstance,
          currentHostContext,
          workInProgress
        );
        appendAllChildren(instance, workInProgress, false, false);
        workInProgress.stateNode = instance;

        // Certain renderers require commit-time effects for initial mount.
        // (eg DOM renderer supports auto-focus for certain elements).
        // Make sure such renderers get scheduled for later work.
        if (
          finalizeInitialChildren(instance, type, newProps, currentHostContext)
        ) {
          markUpdate(workInProgress);
        }
      }
      return null;
    }
  }
}
```

completeWork 跟 beginWork 类似，也是根据 fiber 的类型做不同的处理。这里我们简单看一下 HostComponent 的处理。

- 如果是复用的fiber，已经创建了instance（workInProgress.stateNode !== null），那么就比较两个props，如果有不同就安排副作用，等到提交阶段执行更新。
- 否则，创建instance，如果是DOM render就创建DOM节点。

在completeWork中可能给workInProgress.flags添加新的标记，例如Update标记。

```javascript
function updateHostComponent(
  current: Fiber,
  workInProgress: Fiber,
  type: Type,
  newProps: Props
) {
  // If we have an alternate, that means this is an update and we need to
  // schedule a side-effect to do the updates.
  const oldProps = current.memoizedProps;
  if (oldProps === newProps) {
    // In mutation mode, this is sufficient for a bailout because
    // we won't touch this node even if children changed.
    return;
  }

  // If we get updated because one of our children updated, we don't
  // have newProps so we'll have to reuse them.
  const instance: Instance = workInProgress.stateNode;
  const currentHostContext = getHostContext();
  const updatePayload = prepareUpdate(
    instance,
    type,
    oldProps,
    newProps,
    currentHostContext
  );
  workInProgress.updateQueue = (updatePayload: any);
  // If the update payload indicates that there is a change or if there
  // is a new ref we mark this as an update. All the work is done in commitWork.
  if (updatePayload) {
    markUpdate(workInProgress);
  }
}
```

## 总结

到这里 render 阶段就结束了，这个阶段产生了一个新的 fiber 树，而且某些 fiber 也标记了副作用标记 flags 和 updateQueue。
