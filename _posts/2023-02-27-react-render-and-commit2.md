---
title: "深入渲染和提交"
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

之前我们已经了解了渲染和提交这两个阶段的概要，现在我们更深入了解这两个阶段的细节。

渲染阶段的主要实质性工作在`beginWork | completeWork`这两个函数完成。

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

我们可以看到，在`current !== null`时有可能提前退出 beginWork，我们暂时不深入了解这种情况，毕竟初次渲染时只有`HostRoot.current !== null`。

如果不满足提前退出的条件，那么会根据 fiber 的类型做不同的处理，例如调用`updateHostRoot | updateFunctionComponent | updateClassComponent`，如果类组件还没有实例，可能会在此期间创建一个实例。

### reconcileChildren

多数常见类型的 fiber 会进入`reconcileChildren`函数。

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

我们看到根据判断条件`current === null`有两种情况，`mountChildFibers`和`reconcileChildFibers`实际是调用同一个函数`createChildReconciler`创建，只是参数`shouldTrackSideEffects`不同。

他们的目的是一致的：都是用`nextChildren`产生一个 Fiber 赋给`workInProgress.child`。

<figure>
  <figcaption>reconcileChildren目的</figcaption>
  <img src="/assets/images/reconcileChildren1.png">
</figure>

`createChildReconciler`返回了一个函数`reconcileChildFibers`，请注意与上面的同名变量的区别。

```javascript
function createChildReconciler(
  shouldTrackSideEffects: boolean
): ChildReconciler {
  function reconcileSingleElement() {}

  function reconcileChildrenArray() {}

  function reconcileSingleTextNode() {}

  ...

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
        case REACT_PORTAL_TYPE:
          return placeSingleChild(
            reconcileSinglePortal(
              returnFiber,
              currentFirstChild,
              newChild,
              lanes
            )
          );
        case REACT_LAZY_TYPE:
          const payload = newChild._payload;
          const init = newChild._init;
          // TODO: This function is supposed to be non-recursive.
          return reconcileChildFibers(
            returnFiber,
            currentFirstChild,
            init(payload),
            lanes
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

  return reconcileChildFibers;
}
```

我们先弄清楚`reconcileChildFibers`函数的参数：

- returnFiber，类型是 Fiber，这是当前正在处理的 Fiber，也就是我们常说的 workInProgess
- currentFirstChild，类型是 Fiber 或者 null，这是 returnFiber 在当前 Fiber 树中的那个 Fiber 的第一个子节点
- newChild，类型未知，可能是 React 元素，数组，字符串等等，我们要将 newChild 转换为 Fiber
- lanes，类型是 Lanes，Fiber 中优先级相关内容

在这个函数中会根据`newChild`的类型做不同的处理，例如下面这三种常见类型：

- React 元素类型：调用`reconcileSingleElement`
- 数组：调用`reconcileChildrenArray`
- 字符串，数字：调用`reconcileSingleTextNode`

### reconcileSingleElement

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
      if (elementType === REACT_FRAGMENT_TYPE) {
        if (child.tag === Fragment) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props.children);
          existing.return = returnFiber;
          return existing;
        }
      } else {
        if (child.elementType === elementType) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props);
          existing.ref = coerceRef(returnFiber, child, element);
          existing.return = returnFiber;
          return existing;
        }
      }
      // Didn't match.
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  if (element.type === REACT_FRAGMENT_TYPE) {
    const created = createFiberFromFragment(
      element.props.children,
      returnFiber.mode,
      lanes,
      element.key
    );
    created.return = returnFiber;
    return created;
  } else {
    const created = createFiberFromElement(element, returnFiber.mode, lanes);
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```

上面我们说了 currentFirstChild 可能有三种情况，我们来看看这个函数是如何处理的。

- `currentFirstChild !== null`，比较 key
  - key 相同，比较 type
    - type 相同，使用这个 child 创建 Fiber（调用 createWorkInProgress），**并且删除其他兄弟节点**
    - type 不同，删除这个 child**及其兄弟节点**，并创建新的 Fiber
  - key 不同，删除这个 child
- 否则直接创建新 Fiber

_这中间可能穿插着一些特殊元素类型例如 Fragment，我们不做赘述_

为什么比较 key 是在一个循环中？

答案：因为`currentFirstChild`可能有兄弟节点。

为什么 key 相同情况下要删除多余的兄弟节点？

答案：因为这是在处理单个 React 元素，returnFiber 只能有一个子节点。

### reconcileChildrenArray

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
      // TODO: Move out of the loop. This only happens for the first run.
      resultingFirstChild = newFiber;
    } else {
      // TODO: Defer siblings if we're not at the right index for this slot.
      // I.e. if we had null values before, then we want to defer this
      // for each null value. However, we also don't want to call updateSlot
      // with the previous one.
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  if (newIdx === newChildren.length) {
    // We've reached the end of the new children. We can delete the rest.
    deleteRemainingChildren(returnFiber, oldFiber);
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
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
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
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

数组的协调要复杂一些，我们看到这个过程使用了三次循环。

**第一次循环**

- 取`newChildren[newIdx]`与`oldFiber`（当前 Fiber 树中的节点）调用`updateSlot`返回`newFiber`
- 如果`newFiber === null`跳出循环
- 否则放置 newFiber

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

`updateSlot`中还是靠判断 key 是否匹配来决定是否返回 Fiber（可能新建可能复用），不匹配返回 null。

可以想象为两个数组分别由两个指针从前到后移动处理，key 不匹配就跳出。

如果此次循环能处理完 newChildren 中的所有内容，那么就可以直接返回第一个 childFiber。

**第二次循环（可能发生）**

如果已经没有旧的 Fiber 了，那么 newChildren 中剩余的内容直接创建新的 Fiber。

**第三次循环**

剩余旧的 Fiber 放进 Map 中，索引是 fiber.key 或者 fiber.index。

遍历剩余的 newChidren，如果能找到匹配的旧 Fiber，那么根据情况判断是否能重用，不能就新建 Fiber。
