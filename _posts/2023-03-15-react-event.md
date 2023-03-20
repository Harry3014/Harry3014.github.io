---
title: "react事件系统"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-03-15
last_modified_at: 2023-03-15
categories:
  - React
tags:
  - React
---

此篇文章我们深入分析 react 的事件系统。

## 回到创建 root

在调用 react-dom/client 的 createRoot 时就包含了与事件有关的内容。

```javascript
function createRoot(container, options) {
  // 事件相关
  // 特殊情况：注释标签获取他的父节点
  const rootContainerElement =
    container.nodeType === COMMENT_NODE ? container.parentNode : container;
  listenToAllSupportedEvents(rootContainerElement);
}
```

调用 listenToAllSupportedEvents

```javascript
const listeningMarker = "_reactListening" + Math.random().toString(36).slice(2);

function listenToAllSupportedEvents(rootContainerElement) {
  if (!rootContainerElement[listeningMarker]) {
    rootContainerElement[listeningMarker] = true;
    allNativeEvents.forEach((domEventName) => {
      // We handle selectionchange separately because it
      // doesn't bubble and needs to be on the document.
      if (domEventName !== "selectionchange") {
        if (!nonDelegatedEvents.has(domEventName)) {
          listenToNativeEvent(domEventName, false, rootContainerElement);
        }
        listenToNativeEvent(domEventName, true, rootContainerElement);
      }
    });
    const ownerDocument =
      rootContainerElement.nodeType === DOCUMENT_NODE
        ? rootContainerElement
        : rootContainerElement.ownerDocument;
    if (ownerDocument !== null) {
      // The selectionchange event also needs deduplication
      // but it is attached to the document.
      if (!ownerDocument[listeningMarker]) {
        ownerDocument[listeningMarker] = true;
        listenToNativeEvent("selectionchange", false, ownerDocument);
      }
    }
  }
}
```

allNativeEvents 包含了所有的事件类型，遍历每种事件类型（除了 selectionchange 需要绑定在 document 上）。

nonDelegatedEvents 包含了一些不冒泡的事件，例如 load，resize 等等。不在 nonDelegatedEvents 中的事件需要调用两次 listenToNativeEvent（参数 isCapturePhaseListener=false/true），nonDelegatedEvents 事件需要调用一次（参数 isCapturePhaseListener=true）。

从 isCapturePhaseListener 参数的名称也很容易理解，true 表示在事件**捕获**阶段，false 表示事件**冒泡**阶段。

```javascript
// List of events that need to be individually attached to media elements.
export const mediaEventTypes: Array<DOMEventName> = [
  "abort",
  "canplay",
  "canplaythrough",
  "durationchange",
  "emptied",
  "encrypted",
  "ended",
  "error",
  "loadeddata",
  "loadedmetadata",
  "loadstart",
  "pause",
  "play",
  "playing",
  "progress",
  "ratechange",
  "resize",
  "seeked",
  "seeking",
  "stalled",
  "suspend",
  "timeupdate",
  "volumechange",
  "waiting",
];

// We should not delegate these events to the container, but rather
// set them on the actual target element itself. This is primarily
// because these events do not consistently bubble in the DOM.
export const nonDelegatedEvents: Set<DOMEventName> = new Set([
  "cancel",
  "close",
  "invalid",
  "load",
  "scroll",
  "toggle",
  // In order to reduce bytes, we insert the above array of media events
  // into this Set. Note: the "error" event isn't an exclusive media event,
  // and can occur on other elements too. Rather than duplicate that event,
  // we just take it from the media events array.
  ...mediaEventTypes,
]);
```

我们来看看事件是如何绑定的。

```javascript
function listenToNativeEvent(
  domEventName: DOMEventName,
  isCapturePhaseListener: boolean,
  target: EventTarget
): void {
  let eventSystemFlags = 0;
  if (isCapturePhaseListener) {
    eventSystemFlags |= IS_CAPTURE_PHASE;
  }
  let listener = createEventListenerWrapperWithPriority(
    targetContainer,
    domEventName,
    eventSystemFlags
  );
  if (isCapturePhaseListener) {
    addEventCaptureListener(targetContainer, domEventName, listener);
  } else {
    addEventBubbleListener(targetContainer, domEventName, listener);
  }
}
```

创建事件监听器，不同的事件分为三大类，分别对应不同的事件监听器。

```javascript
export function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags
): Function {
  const eventPriority = getEventPriority(domEventName);
  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:
      listenerWrapper = dispatchDiscreteEvent;
      break;
    case ContinuousEventPriority:
      listenerWrapper = dispatchContinuousEvent;
      break;
    case DefaultEventPriority:
    default:
      listenerWrapper = dispatchEvent;
      break;
  }
  return listenerWrapper.bind(
    null,
    domEventName,
    eventSystemFlags,
    targetContainer
  );
}
```

然后分为捕获和冒泡两张情况注册事件监听器。

```javascript
export function addEventBubbleListener(
  target: EventTarget,
  eventType: string,
  listener: Function
): Function {
  target.addEventListener(eventType, listener, false);
  return listener;
}

export function addEventCaptureListener(
  target: EventTarget,
  eventType: string,
  listener: Function
): Function {
  target.addEventListener(eventType, listener, true);
  return listener;
}
```

_可能还会传入 passive 参数的情况，这里不做讨论_

_passive=true 表示事件监听器不会掉用 preventDefault，否则抛出控制台警告_

### 小结

- 在创建 root 时，所有事件绑定在 root container 上
- 部分事件只处理捕获阶段

## 事件处理

之前创建的事件监听器，无论是哪一种，他们的核心都是 dispatchEvent 函数

```javascript
export function dispatchEvent(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget,
  nativeEvent: AnyNativeEvent
): void {}
```

这里我们跳过 dispatchEvent 的中间的一些内容，最后都会进入 dispatchEventsForPlugins 函数。

```javascript
function dispatchEventsForPlugins(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  nativeEvent: AnyNativeEvent,
  targetInst: null | Fiber,
  targetContainer: EventTarget
): void {
  // 获取触发事件的元素
  const nativeEventTarget = getEventTarget(nativeEvent);
  const dispatchQueue: DispatchQueue = [];
  extractEvents(
    dispatchQueue,
    domEventName,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags,
    targetContainer
  );
  processDispatchQueue(dispatchQueue, eventSystemFlags);
}
```

dispatchQueue 是很重要的一个变量。

- 调用 extractEvents 向 dispatchQueue 中添加内容
- 调用 processDispatchQueue 处理 dispatchQueue

```javascript
function extractEvents(
  dispatchQueue: DispatchQueue,
  domEventName: DOMEventName,
  targetInst: null | Fiber,
  nativeEvent: AnyNativeEvent,
  nativeEventTarget: null | EventTarget,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget
): void {
  // 转换得到react事件处理名称，例如click -> onClick
  const reactName = topLevelEventsToReactNames.get(domEventName);
  if (reactName === undefined) {
    return;
  }
  let SyntheticEventCtor = SyntheticEvent;
  let reactEventType: string = domEventName;
  // 根据dom事件类型创建react内部的合成事件类型
  switch (domEventName) {
    case "keydown":
    case "keyup":
      SyntheticEventCtor = SyntheticKeyboardEvent;
      break;
    default:
      // Unknown event. This is used by createEventHandle.
      break;
  }

  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;

  // Some events don't bubble in the browser.
  // In the past, React has always bubbled them, but this can be surprising.
  // We're going to try aligning closer to the browser behavior by not bubbling
  // them in React either. We'll start by not bubbling onScroll, and then expand.
  const accumulateTargetOnly =
    !inCapturePhase &&
    // TODO: ideally, we'd eventually add all events from
    // nonDelegatedEvents list in DOMPluginEventSystem.
    // Then we can remove this special list.
    // This is a breaking change that can wait until React 18.
    domEventName === "scroll";

  const listeners = accumulateSinglePhaseListeners(
    targetInst,
    reactName,
    nativeEvent.type,
    inCapturePhase,
    accumulateTargetOnly,
    nativeEvent
  );
  if (listeners.length > 0) {
    // Intentionally create event lazily.
    const event: ReactSyntheticEvent = new SyntheticEventCtor(
      reactName,
      reactEventType,
      null,
      nativeEvent,
      nativeEventTarget
    );
    dispatchQueue.push({ event, listeners });
  }
}
```

extractEvents 函数的核心是 accumulateSinglePhaseListeners。

```javascript
export function accumulateSinglePhaseListeners(
  targetFiber: Fiber | null,
  reactName: string | null,
  nativeEventType: string,
  inCapturePhase: boolean,
  accumulateTargetOnly: boolean,
  nativeEvent: AnyNativeEvent
): Array<DispatchListener> {
  const captureName = reactName !== null ? reactName + "Capture" : null;
  const reactEventName = inCapturePhase ? captureName : reactName;
  let listeners: Array<DispatchListener> = [];

  let instance = targetFiber;
  let lastHostComponent = null;

  // Accumulate all instances and listeners via the target -> root path.
  while (instance !== null) {
    const { stateNode, tag } = instance;
    // Handle listeners that are on HostComponents (i.e. <div>)
    if (tag === HostComponent && stateNode !== null) {
      lastHostComponent = stateNode;

      // Standard React on* listeners, i.e. onClick or onClickCapture
      if (reactEventName !== null) {
        const listener = getListener(instance, reactEventName);
        if (listener != null) {
          listeners.push(
            createDispatchListener(instance, listener, lastHostComponent)
          );
        }
      }
    }
    // If we are only accumulating events for the target, then we don't
    // continue to propagate through the React fiber tree to find other
    // listeners.
    if (accumulateTargetOnly) {
      break;
    }
    instance = instance.return;
  }
  return listeners;
}
```

从 targetFiber 开始向上遍历直到 root，使用 reactEventName 去获取 listener。

```javascript
export default function getListener(
  inst: Fiber,
  registrationName: string
): Function | null {
  const stateNode = inst.stateNode;
  if (stateNode === null) {
    // Work in progress (ex: onload events in incremental mode).
    return null;
  }
  const props = getFiberCurrentPropsFromNode(stateNode);
  if (props === null) {
    // Work in progress.
    return null;
  }
  const listener = props[registrationName];
  if (shouldPreventMouseEvent(registrationName, inst.type, props)) {
    return null;
  }

  if (listener && typeof listener !== "function") {
    throw new Error(
      `Expected \`${registrationName}\` listener to be a function, instead got a value of \`${typeof listener}\` type.`
    );
  }

  return listener;
}
```

获取 listener 是去 props 中获取，就是我们创建 react 元素时传递的属性。

接下来就是处理 listener 的调用了。

```javascript
// 依次处理dispatchQueue
export function processDispatchQueue(
  dispatchQueue: DispatchQueue,
  eventSystemFlags: EventSystemFlags
): void {
  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;
  for (let i = 0; i < dispatchQueue.length; i++) {
    const { event, listeners } = dispatchQueue[i];
    processDispatchQueueItemsInOrder(event, listeners, inCapturePhase);
    //  event system doesn't use pooling.
  }
  // This would be a good time to rethrow if any of the event handlers threw.
  rethrowCaughtError();
}

function processDispatchQueueItemsInOrder(
  event: ReactSyntheticEvent,
  dispatchListeners: Array<DispatchListener>,
  inCapturePhase: boolean
): void {
  let previousInstance;
  if (inCapturePhase) {
    // 捕获阶段需要倒序处理
    for (let i = dispatchListeners.length - 1; i >= 0; i--) {
      const { instance, currentTarget, listener } = dispatchListeners[i];
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  } else {
    for (let i = 0; i < dispatchListeners.length; i++) {
      const { instance, currentTarget, listener } = dispatchListeners[i];
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  }
}
```

excuteDispatch 就是关键的调用了。

```javascript
function executeDispatch(
  event: ReactSyntheticEvent,
  listener: Function,
  currentTarget: EventTarget
): void {
  const type = event.type || "unknown-event";
  event.currentTarget = currentTarget;
  // 会调用listener，在此不详细分析
  invokeGuardedCallbackAndCatchFirstError(type, listener, undefined, event);
  event.currentTarget = null;
}
```

- 参数 event 是 react 的合成事件
- 参数 listener 是开发者传递给 react 元素的事件监听器
- currentTarget 是当前的事件处理目标

### 小结

触发事件后，进入绑定在 root container 上的事件监听器。从触发事件的 fiber 开始向上遍历，获取所有匹配的 listener，然后执行他们。

## 总结

我们可以看到，所有的事件监听器就是绑定在 root container 上的。

**请注意：这种处理方式并不全是事件委托**。因为并不是所有事件都支持冒泡。

对于不支持冒泡的事件，react 利用了事件流的捕获。
