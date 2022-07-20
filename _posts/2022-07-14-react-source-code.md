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

## React 元素

`root.render`方法接受的参数是 React 元素，React 元素的大致结构如下，这里还有很多属性没有列出，例如`$$typeof`，`key`，`ref`等等。

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

## 渲染

在调用`root.render`后，React 并没有立即执行渲染的相关操作，而是把渲染当做是一个任务放进任务队列，然后由 Scheduler 来调度任务。

### React 任务调度

React 有两个任务队列`taskQueue`和`timeQueue`，`timeQueue`是用于存放延时任务。这两个任务队列都是最小堆，那为什么要使用最小堆呢？因为每个任务都有优先级，需要让优先级高的任务先执行。

#### 任务优先级

React 的任务分为 5 个优先级，由高到低依次是：

1. 立即执行
2. 用户阻塞
3. 普通优先级
4. 低优先级
5. 空闲

#### 执行任务时机

现在任务队列中已经存在任务了，那什么时候可以执行任务呢？

执行任务就是调用任务的回调函数，这里 render 的回调函数是`performConcurrentWorkOnRoot`。

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
