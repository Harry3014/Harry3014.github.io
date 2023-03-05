---
title: "深入React前的准备"
excerpt: ""
toc: true
toc_sticky: true
date: 2022-08-23
last_modified_at: 2022-08-23
categories:
  - React
tags:
  - React
---

我们将在这篇文章中讨论一些 React 的基础知识，以便我们后面深入学习 React。

## 声明式

React 使用声明式编写 UI，这使得开发者的工作变得更加容易。

在 React 的<a href="https://beta.reactjs.org/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative" target="_blank">文档</a>中举了一个例子来比较声明式与指令式。

你坐上一辆车需要到某个目的地：

<figure>
  <figcaption>声明式：直接告诉司机目的地，他会自动把你送到目的地</figcaption>
  <img src="/assets/images/i_declarative-ui-programming.png">
</figure>

<figure>
  <figcaption>指令式：告诉司机什么时候直行，什么时候转向。</figcaption>
  <img src="/assets/images/i_imperative-ui-programming.png">
</figure>

## 组件 & 元素

组件：宿主组件，函数组件，类组件等，是可复用的 UI 片段。

元素：React 在内部创建用于描述 UI 的对象，可以调用`React.createElement`或者 JSX 创建。这里有一个很重要的点需要了解：应该把元素当成不可变的对象，只要一经创建就不可修改。

元素对象包含下面一些属性：

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react/src/ReactElement.js#L148" target="_blank">跳转到 Github 查看源码</a>。

- \$\$typeof

  它的<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/shared/ReactSymbols.js#L15" target="_blank">值</a>是 symbol 类型，用于标识这个 object 是 React 元素。

- type

  它的值是`string | function | class`，分别对应宿主组件，函数组件和类组件。

- key

  给 React 元素添加一个唯一标识可以帮助 React 区分同一个列表中的元素，这对于添加子项，删除子项，给子项排序是非常重要的。

- props

  保存了元素的一些属性，`props.children`中包含了子元素。

- ref

举个例子，`<h1 className="greeting">Hello from <i>React</i></h1>`会创建一个元素：

```javascript
{
  // 省略$$typeof属性
  type: "h1",
  props: {
    className: "greeting",
    children: [
      "Hello from ",
      {
        $$typeof: Symbol(react.element),
        type: "i",
        props: {
          children: "React"
        }
        key: null,
        ref: null,
      }
    ]
  },
  key: null,
  ref: null
}
```

### JSX

JSX 依靠 Babel 等工具转换为 JavaScript。在 React17 以前，JSX 会被转换为`React.createElement`，这也是为什么必须要引入 react package 的原因。

假设源代码如下：

```jsx
import React from "react";

function App() {
  return <h1>Hello world</h1>;
}
```

旧的 JSX 转换会转换成下面的结果：

```javascript
import React from "React";

function App() {
  return React.createElement("h1", null, "Hello world");
}
```

新的 JSX 可以不用引入 react package，会转换为下面的结果：

```jsx
function App() {
  return <h1>Hello world</h1>;
}
```

## 跨平台

React 支持跨平台，因为 React 有不同的渲染器，不同的渲染器在不同的平台上可以将组件渲染成不同的内容，比如：

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-dom" target="_blank">react-dom</a>渲染成 DOM。

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-native-renderer" target="_blank">react-native-renderer</a>渲染成 Native 视图。

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-test-renderer" target="_blank">react-test-renderer</a>渲染成 JSON 树。

- <a href="https://github.com/facebook/react/tree/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-art" target="_blank">react-art</a>渲染成矢量图。

虽然不同的渲染器有很大差异，但也需要共享一些逻辑，特别是协调算法需要尽可能相似，这样才能保持跨平台工作一致，协调算法的部分称为 reconciler。

<figure>
  <figcaption>reconciler & renderer</figcaption>
  <img src="/assets/images/react_reconciler_renderer.jpg">
</figure>

## 协调

上面我们提到了协调，那么协调是什么呢？

协调就是比较即将需要渲染的内容与上一次渲染有哪些不同。

### Stack reconciler

Stack reconciler 是 React 15 及更早的解决方案。Stack reconciler 通过传入的元素创建了一些实例，然后再创建 DOM，更新的时候更新实例，然后更新 DOM。它的处理方式一层一层向下的，先处理本元素，然后处理子元素，函数组件和类组件的 render 方法返回新的元素。

> <a href="https://zh-hans.reactjs.org/docs/implementation-notes.html" target="_blank">React 文档</a>中有 Stack reconciler 的简单实现，有兴趣可以阅读。

<figure>
  <figcaption>Stack reconciler</figcaption>
  <img src="/assets/images/stack_reconciler.png">
</figure>

_之所以叫 Stack reconciler，是因为它的处理方式类似于调用栈，而非官方名称_

Stack reconciler 有很明显的缺点：同步无法中断，如果中断了，这时浏览器需要重新绘制，那么可能导致 UI 不一致。例如下图中处理完第二个 item 后中断，这时候第一个 item 和第二个 item 的 DOM 已经发生变化了。

<figure>
  <img src="/assets/images/react_element_instance_dom.jpg">
</figure>

### Fiber reconciler

为了解决 Stack reconciler 的问题，React16 提出了 Fiber reconciler。

Fiber reconciler 将可中断的任务切分成更小的任务单元，在执行完一个任务单元后可以让出主线程，如果有更高优先级的任务可以在这时候执行。

<figure>
  <figcaption>Fiber reconciler</figcaption>
  <img src="/assets/images/fiber_reconciler1.png">
  <img src="/assets/images/fiber_reconciler2.png">
</figure>

而且 Fiber reconciler 分成渲染和提交两个阶段，渲染阶段只是计算两次渲染的差异，在提交阶段统一提交到 DOM，这样就保证 UI 的一致性。渲染阶段可中断去执行其他任务，但是在提交阶段无法中断，否则无法保证 UI 的一致性。

<figure>
  <img src="/assets/images/react_phases.png">
</figure>

**Fiber**

Fiber reconciler 中的一个核心概念就是 Fiber，那么 Fiber 是什么呢？Fiber 就是任务切分出来的一个个小的任务单元，你可以把它理解成原来 Stack 中的一帧，它受 React 控制：

- 可以暂停（pause），然后回来继续执行未完成的任务
- 可以中止（abort），注意与暂停的区别，这里指放弃掉
- 可以赋予优先级
- 可以重用

> 更多 Fiber 的目标可以在<a href="https://zh-hans.reactjs.org/docs/codebase-overview.html#fiber-reconciler" target="_blank">React 文档</a>中查看

Fiber 对象包含下列属性：

> <a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactInternalTypes.js#L79" target="_blank">跳转到 Github 查看源码</a>

- tag

  定义了 Fiber 的类型，<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactWorkTags.js" target="_blank">源码</a>中定义了很多种类型，例如下面这几个常见类型:

  ```javascript
  export const FunctionComponent = 0;
  export const ClassComponent = 1;
  export const IndeterminateComponent = 2; // Before we know whether it is function or class
  export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
  export const HostComponent = 5;
  ```

- key and type

  key 和 type 在创建 Fiber 时都是从元素直接<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiber.js#L650" target="_blank">复制</a>过来的，`fiber.key = element.key; fiber.type = element.type`。

  函数组件和类组件的`type`就是他们自己，宿主组件的`type`是`string`，例如`div`, `span`。

- stateNode

  维护了 Fiber 的本地状态，例如 DOM 节点或者类组件的实例等等。

- return, child and sibling

  这三个属性使得不同的 Fiber 之间建立起了联系，`fiber.child`指向第一个子节点，`fiber.return`指向父节点，`fiber.sibling`指向下一个兄弟节点。

- pendingProps and memoizedProps

  `pendingProps`是执行 Fiber 前设置的属性，`memoizedProps`是执行 Fiber 后设置的属性。如果二者相同，那么就表示上一次 Fiber 的输出可以重用，避免重复工作。

- alternate

  一个组件可能不止有一个 Fiber：`current` fiber 表示当前状态，`workInProgress` fiber 表示正在处理。`current.alternate === workInProgress`并且`workInProgress.alternate === current`。

**Fiber tree**

Fiber 中的属性`return, child, sibling`使得 Fiber 之间建立了联系，构成了 Fiber 树，根结点叫做`HostRoot`。

<figure>
  <figcaption>Fiber tree</figcaption>
  <img src="/assets/images/fiber_structure.png">
</figure>

**双缓冲**

React 在内部维护了两个版本的 Fiber Tree：

- current：已处理完成的版本
- workInProgress：正在处理的版本

它们之间使用`alternate`属性关联，这样做可以重用 Fiber 对象减少内存分配。

<a href="https://github.com/facebook/react/blob/855b77c9bbee347735efcd626dda362db2ffae1d/packages/react-reconciler/src/ReactFiber.js#L266" target="_blank">源码</a>中创建`workInProgress`就体现了 React 使用了双缓冲解决方案。

## Virtual DOM

React 推出来以来，Virtual DOM 被认为是它的“杀手锏”。元素和 Fiber 被认为是 Virtual DOM 实现的一部分。在 React 的<a href="https://zh-hans.reactjs.org/docs/faq-internals.html#what-is-the-virtual-dom" target="_blank">文档</a>由更多关于 Virtual DOM 的描述。

_其实叫 Virtul DOM 并不十分贴切，因为 React 并不是只能渲染为 DOM_
