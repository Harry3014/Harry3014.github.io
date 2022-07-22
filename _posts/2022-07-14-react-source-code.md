---
title: "React源码分析"
excerpt: ""
toc: true
date: 2022-07-14
last_modified_at: 2022-07-14
categories:
  - React
tags:
  - React
---

本篇文章都是基于<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a" target="_blank">React 18</a>。

## React 源码的架构

React 的源码的核心主要在下面这几个包。

### React Core

<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react" target="_blank">React 核心包</a>包含了定义 React 组件的 API，例如`React.createElement`，它不依赖与任何平台。

### Renderer

渲染器管理了一棵由 React 元素组成的树，然后在不同的平台将这棵树渲染成不同的内容。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-dom" target="_blank">React DOM Renderer</a>渲染成 DOM。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-native-renderer" target="_blank">React Native Renderer</a>渲染成 Native 视图。

- <a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-test-renderer" target="_blank">React Test Renderer</a>渲染成 JSON 树。

### Reconciler

前面我们说的渲染器管理了一棵 React 元素树，当组件`props`或者`state`变化时又生成了一棵新的树，如果直接使用新的树渲染，那就太浪费资源了。<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-reconciler" target="_blank">协调器</a>会这两棵树之间的差别，然后交给渲染器去高效的更新变化的部分。

### Scheduler

React 的设计理念中会把工作切成一个个小的任务，而且每个任务有优先级。<a href="https://github.com/facebook/react/tree/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler" target="_blank">调度器</a>就是合理的安排每个任务何时执行，何时能暂停任务，暂停后何时继续完成任务。

## 从一个简单的例子开始

```jsx
const element = (
  <div>
    <h1 className="header">Hello, world!</h1>
    <h2 style={{ textAlign: "right" }}>From future</h2>
  </div>
);
const container = document.getElementById("root");
const root = ReactDOM.createRoot(container);
root.render(element);
```

### React 元素

上面的 JSX 会转换成 React 元素，React 元素的大致结构如下，这里还有很多属性没有列出，例如`$$typeof`，`key`，`ref`等等，<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react/src/ReactElement.js#L148" target="_blank">源码位置</a>。

```javascript
const element = {
  type: "div",
  props: {
    children: [
      {
        type: "h1",
        props: {
          className: "header",
          children: "hello world",
        },
      },
      {
        type: "h2",
        props: {
          style: {
            textAlign: "right",
          },
          children: "from future",
        },
      },
    ],
  },
};
```

## React 任务调度

在调用`root.render`后，React 并没有立即执行渲染的相关操作，而是把渲染当做是一个任务放进任务队列，然后由 Scheduler 来调度任务。

React 有两个任务队列`taskQueue`和`timeQueue`，`timeQueue`是用于存放延时任务。这两个任务队列都是<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/SchedulerMinHeap.js" target="_blank">优先队列</a>，为什么要使用优先队列呢？因为每个任务都有优先级，需要让优先级高的任务先执行。

### 任务优先级

React 的任务分为 5 个优先级，由高到低依次是：

1. ImmediatePriority 立即执行
2. UserBlockingPriority 用户阻塞
3. NormalPriority 普通优先级
4. LowPriority 低优先级
5. IdlePriority 空闲

### 执行任务

注册任务时，执行任务就是调用任务的回调函数，这里 render 的回调函数是`performConcurrentWorkOnRoot`。

## 协调

上一步我们已经准备开始执行 render root 这个任务，这里我们要谈到 React 的[Fiber Reconciler](https://zh-hans.reactjs.org/docs/codebase-overview.html)，在官方文档的简介中，Fiber Reconciler 的一个很重要的目的就是：能够把可中断的任务切片处理。

源码中`workInProgressRoot`表示正在处理的 fiber root，`workInProgrss`表示正在处理的 fiber。

render 会调用下面这样一个函数进入工作循环。

```javascript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```
