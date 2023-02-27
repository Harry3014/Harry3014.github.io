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
      if (matchSomeCondition) {
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
