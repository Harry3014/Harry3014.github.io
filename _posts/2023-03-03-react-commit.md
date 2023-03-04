---
title: "深入提交阶段"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-03-03
last_modified_at: 2023-03-03
categories:
  - React
tags:
  - React
---

提交阶段可以分为三个子阶段：

1. before mutation 阶段
2. mutation 阶段
3. layout 阶段

## before mutation 阶段

在调用 commitBeforeMutationEffects 处 React 的源码注释如下：

```javascript
// The first phase a "before mutation" phase. We use this phase to read the
// state of the host tree right before we mutate it. This is where
// getSnapshotBeforeUpdate is called.
const shouldFireAfterActiveInstanceBlur = commitBeforeMutationEffects(
  root,
  finishedWork
);
```

在此阶段可以在修改 host tree 前读取他的状态，这也是调用 getSnapshotBeforeUpdate 声明周期函数的地方。

```javascript
function commitBeforeMutationEffects(
  root: FiberRoot,
  firstChild: Fiber
): boolean {
  nextEffect = firstChild;
  commitBeforeMutationEffects_begin();
}

function commitBeforeMutationEffects_begin() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const child = fiber.child;
    if (
      (fiber.subtreeFlags & BeforeMutationMask) !== NoFlags &&
      child !== null
    ) {
      child.return = fiber;
      nextEffect = child;
    } else {
      commitBeforeMutationEffects_complete();
    }
  }
}

function commitBeforeMutationEffects_complete() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    try {
      commitBeforeMutationEffectsOnFiber(fiber);
    } catch (error) {
      captureCommitPhaseError(fiber, fiber.return, error);
    }

    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }

    nextEffect = fiber.return;
  }
}

function commitBeforeMutationEffectsOnFiber(finishedWork: Fiber) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;

  switch (finishedWork.tag) {
    case ClassComponent: {
      if ((flags & Snapshot) !== NoFlags) {
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;

          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState
          );
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
      }
      break;
    }
  }
}
```

这个过程从HostRoot开始遍历，如果fiber.subtreeFlags完全匹配不上BeforeMutationMask，那么就可以commitBeforeMutationEffectsOnFiber，这个函数中也是根据fiber类型做不同的处理，ClassComponent会调用getSnapshotBeforeUpdate声明周期函数。

处理完本fiber后，判断sibling是否需要处理，sibling处理完成后返回parent，逻辑与completeUnitOfWork类似。

## mutation 阶段

## layout 阶段
