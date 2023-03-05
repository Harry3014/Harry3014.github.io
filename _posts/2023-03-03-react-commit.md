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

调用 commitBeforeMutationEffects 函数进入此阶段：

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

这个过程从 HostRoot 开始遍历，如果 fiber.subtreeFlags 完全匹配不上 BeforeMutationMask，那么就可以 commitBeforeMutationEffectsOnFiber，这个函数中也是根据 fiber 类型做不同的处理，ClassComponent 会调用 getSnapshotBeforeUpdate 声明周期函数。

处理完本 fiber 后，判断 sibling 是否需要处理，sibling 处理完成后返回 parent，逻辑与 completeUnitOfWork 类似。

## mutation 阶段

调用 commitMutationEffects 函数进入此阶段。

```javascript
// The next phase is the mutation phase, where we mutate the host tree.
commitMutationEffects(root, finishedWork, lanes);
```

此阶段修改 host tree，DOM render 就是修改 DOM 树。

commitMutationEffects的核心就是调用 commitMutationEffectsOnFiber 函数。

<figure>
<img src="/assets/images/commitMutationEffectsOnFiber.jpg" />
</figure>

commitMutationEffectsOnFiber 根据 fiber 的类型做不同处理，但是几乎每种 fiber 都会先调用下面这两行代码。

```javascript
recursivelyTraverseMutationEffects(root, finishedWork, lanes);
commitReconciliationEffects(finishedWork);
```

从函数名称也很容易很理解：递归遍历，然后提交协调副作用。

```javascript
function recursivelyTraverseMutationEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes
) {
  // Deletions effects can be scheduled on any fiber type. They need to happen
  // before the children effects hae fired.
  const deletions = parentFiber.deletions;
  if (deletions !== null) {
    for (let i = 0; i < deletions.length; i++) {
      const childToDelete = deletions[i];
      try {
        commitDeletionEffects(root, parentFiber, childToDelete);
      } catch (error) {
        captureCommitPhaseError(childToDelete, parentFiber, error);
      }
    }
  }

  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      commitMutationEffectsOnFiber(child, root, lanes);
      child = child.sibling;
    }
  }
}
```

**删除副作用**

在递归遍历 child 和 sibling 之前，先提交删除副作用。

提交删除副作用的过程也是递归的，因为删除的 fiber 可能有 children。<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCommitWork.js#L1933" target="_blank">查看 commitDeletionEffects 源码</a>

删除时主要做下面内容：

- 删除 host node，DOM render 调用 removeChild 删除 DOMe 节点，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCommitWork.js#L2090" target="_blank">查看源码</a>
- 拆卸 refs
- 清除 layout effect，即调用 useLayoutEffect 返回的 clean up 函数（函数组件的 Hook），<a href="<https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCommitWork.js#L2192" target="_blank">查看源码</a>
- 类组件调用 componentWillUnmount 生命周期函数，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCommitWork.js#L2235" target="_blank">查看源码</a>

**协调副作用**

我们在渲染阶段中提到过，协调阶段涉及的 fiber 操作只有创建、移动、删除，上面我们已经处理了删除，现在处理创建和移动，这两者实际是一种类型，都是 Placement。

```javascript
function commitReconciliationEffects(finishedWork: Fiber) {
  // Placement effects (insertions, reorders) can be scheduled on any fiber
  // type. They needs to happen after the children effects have fired, but
  // before the effects on this fiber have fired.
  const flags = finishedWork.flags;
  if (flags & Placement) {
    try {
      commitPlacement(finishedWork);
    } catch (error) {
      captureCommitPhaseError(finishedWork, finishedWork.return, error);
    }
    // Clear the "placement" from effect tag so that we know that this is
    // inserted, before any life-cycles like componentDidMount gets called.
    // TODO: findDOMNode doesn't rely on this any more but isMounted does
    // and isMounted is deprecated anyway so we should be able to kill this.
    finishedWork.flags &= ~Placement;
  }
}
```

DOM render 调用 insertBefore，appendChild 操作 DOM 节点。

**其他副作用**

例如：

- FunctionComponent 在 Update 时，销毁layout effect，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCommitWork.js#L2542" target="_blank">查看源码</a>
- HostComponent 在 Update 时修改 host node，例如 DOM render 修改 DOM 节点的 style，设置文本内容以及其他属性，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiberCommitWork.js#L2653" target="_blank">查看源码</a>

### mutation 阶段完成后

mutation 阶段完成后，可以将新的 fiber 树替换到 root.current 上了。

```javascript
// The work-in-progress tree is now the current tree. This must come after
// the mutation phase, so that the previous tree is still current during
// componentWillUnmount, but before the layout phase, so that the finished
// work is current during componentDidMount/Update.
root.current = finishedWork;
```

## layout 阶段

调用 commitLayoutEffects 函数进入此阶段。

```javascript
// The next phase is the layout phase, where we call effects that read
// the host tree after it's been mutated. The idiomatic use case for this is
// layout, but class component lifecycles also fire here for legacy reasons.
commitLayoutEffects(finishedWork, root, lanes);
```

与mutation阶段类似，commitLayoutEffects 的核心是调用commitLayoutEffectOnFiber函数。

<figure>
<img src="/assets/images/commitLayoutEffectOnFiber.jpg" />
</figure>

commitLayoutEffectOnFiber也是根据fiber的类型做不同处理，处理方式也是递归遍历（调用recursivelyTraverseLayoutEffects）。

**FunctionComponent**

在Update时，挂载layout effect。

**ClassComponent**

在Update时，挂载时调用componentDidMount，更新时调用componentDidUpdte。

有一些fiber还会绑定ref。

## 三个阶段之外

到这里三个阶段就结束了，但是在一个阶段前和最后一个阶段后，还有一个可以讨论的内容。

```javascript
// If there are pending passive effects, schedule a callback to process them.
  // Do this as early as possible, so it is queued before anything else that
  // might get scheduled in the commit phase. (See #16714.)
  // TODO: Delete all other places that schedule the passive effect callback
  // They're redundant.
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      pendingPassiveEffectsRemainingLanes = remainingLanes;
      // workInProgressTransitions might be overwritten, so we want
      // to store it in pendingPassiveTransitions until they get processed
      // We need to pass this through as an argument to commitRoot
      // because workInProgressTransitions might have changed between
      // the previous render and commit if we throttle the commit
      // with setTimeout
      pendingPassiveTransitions = transitions;
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        // This render triggered passive effects: release the root cache pool
        // *after* passive effects fire to avoid freeing a cache pool that may
        // be referenced by a node in the tree (HostRoot, Cache boundary etc)
        return null;
      });
    }
  }
```

```javascript
// If the passive effects are the result of a discrete render, flush them
  // synchronously at the end of the current task so that the result is
  // immediately observable. Otherwise, we assume that they are not
  // order-dependent and do not need to be observed by external systems, so we
  // can wait until after paint.
  if (includesSyncLane(pendingPassiveEffectsLanes) && root.tag !== LegacyRoot) {
    flushPassiveEffects();
  }
```

在三个阶段前后都调用了flushPassiveEffects，这主要是useEffect的处理。

在commit前schedule了这个任务，但是在commit后，如果passive effect是同步的，应该在这里处理。
