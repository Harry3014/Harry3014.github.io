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

`mountChildFibers`和`reconcileChildFibers`实际是调用同一个函数`createChildReconciler`创建，只是参数`shouldTrackSideEffects`不同。

`createChildReconciler`返回了一个函数`reconcileChildFibers`，请注意与上面的同名变量的区别。

根据 child 的类别做不同的处理，比如元素，数组，字符串和数字等等。

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
