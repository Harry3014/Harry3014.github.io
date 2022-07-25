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
const style = { textAlign: "right" };
const element = (
  <div>
    <h1 className="header">Hello, world!</h1>
    <h2 style={style}>From future</h2>
  </div>
);
const container = document.getElementById("root");
const root = ReactDOM.createRoot(container);
root.render(element);
```

## React 元素

上面的 JSX 会转换成 React 元素，React 元素的大致结构如下，这里还有很多属性没有列出，例如`$$typeof`，`key`，`ref`等等，<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react/src/ReactElement.js#L148" target="_blank">源码</a>查看完整结构。

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

## ReactDOM.createRoot

<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-dom/src/client/ReactDOMRoot.js#L167" target="_blank">ReactDOM.createRoot</a>返回 ReactDOMRoot 对象。

## React 任务调度

在调用<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/react-dom/src/client/ReactDOMRoot.js#L93" target="_blank">root.render</a>后，React 并没有立即执行渲染，而是把渲染函数当成回调函数交给调度器去安排。

调度器接收到一个<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L308" target="_blank">安排请求</a>时，会根据提供的优先级创建一个新的任务，然后将这个任务放进任务队列中等待后续的安排。

React 有<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L72" target="_blank">两个任务队列</a>`taskQueue`和`timeQueue`，`timeQueue`是用于存放延时任务，这两个任务队列都是用<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/SchedulerMinHeap.js" target="_blank">最小堆</a>实现的优先队列。

### 任务优先级

上面的任务队列为什么要使用优先队列呢？因为每个任务都有优先级，需要让优先级高的任务先执行。

React 的任务分为 5 个优先级，由高到低依次是：

1. ImmediatePriority 立即执行
2. UserBlockingPriority 用户阻塞
3. NormalPriority 普通优先级
4. LowPriority 低优先级
5. IdlePriority 空闲

<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/SchedulerPriorities.js" target="_blank">源码</a>中声明了 6 个优先级，但是<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L24" target="_blank">实际</a>只使用了 5 个。

### 进入工作循环

把任务放进任务队列后，如果满足执行任务的<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L381" target="_blank">条件</a>，那么就可以准备执行任务了。

在浏览器环境中如果支持`MessageChannel`，那么可以就可以通过`MessagePort.postMessage`发送一条消息，然后`MessagePort.onMessage`在收到消息后进入工作循环，如果不支持，那么就通过`setTimeout`安排一个 0 秒后的定时任务。<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L569">源码</a>中说明了为什么优先使用`MessageChannel`，因为在<a href="https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers" target="_blank">HTML5 标准</a>中注明了在嵌套了 5 次定时任务后，至少会有 4ms 的延时。

进入工作循环后从任务队列中取出优先级最高的任务准备执行，为什么说准备执行呢？因为真正执行任务是要满足一些<a href="https://github.com/facebook/react/blob/3ddbedd0520a9738d8c3c7ce0268542e02f9738a/packages/scheduler/src/forks/Scheduler.js#L197" target="_blank">条件</a>的，如果任务过期或者执行任务已经超过了一个限制时间，那么应该把主线程的控制权让给更优先的任务，例如浏览器绘制，用户输入等等。

## 协调
