---
title: "React底层原理"
excerpt: ""
toc: true
date: 2022-05-28
last_modified_at: 2022-05-28
categories:
  - React
tags:
  - React
---

在了解 React 底层原理之前，我们先来看看 React 与传统的面向对象编程有什么不同。

## React 与面向对象编程的不同

面向对象编程需要你自己管理组件的实例，包括创建、更新、销毁等等，父组件可能直接保存子组件的实例，这使得组件之间变得更难解耦。

React 不用自己直接参与创建组件实例，React 会负责这一部分。而且也只有类组件才有实例，函数组件没有实例。

_[参考 React 的这篇博客](https://zh-hans.reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)_

## 声明式和指令式编程的不同

JSX 是一种声明式编写 UI 的方式，与指令式相比，不需要自己来控制 UI 的变化，只需要告诉 React 我们希望看到什么状态的 UI，React 会自行控制 UI 应该怎样变化。我们可以想象一个场景：乘客坐上了一辆出租车，声明式就需要乘客自己告诉司机应该怎样走，向左向右等等，而声明式只需要告诉司机目的地，无论怎样走，司机会把你带到目的地，React 就类似司机这个角色。

_[参考 React 官方文档](https://beta.reactjs.org/learn/reacting-to-input-with-state)_

## React 的基础：元素

元素 element 是 React 的基础，是用来描述组件或者原生 DOM 元素的一个不可变的 JS 对象。

_不可变并不是指 element 的属性不能变，而是 element 不能再被赋值为其他值，可以把 element 理解为 const 声明_

### 元素结构

element 的结构大致如下，[源码链接](https://github.com/facebook/react/blob/main/packages/react/src/ReactElement.js#L148)

```js
{
  $$typeof: Symbol.for("react element"),
  type,
  key,
  ref,
  props,
}
```

出于[安全原因](https://github.com/facebook/react/pull/4832)，添加了`$$typeof`属性。

`type`属性可能是`string`表示 DOM 元素，也可能是函数或者类表示组件元素。

`props`中的`children`属性可能包含一个或者多个子元素，也可能没有子元素。

这样的结构就组成了一个元素树（element tree），元素既不是真实的 DOM，也不是真实的组件实例，但它完全能够描述 DOM 和组件实例。元素比 DOM 的要轻很多，它只是一个 object。这种结构使得遍历非常的简单，也可以随意创建丢弃，因为它并不代表屏幕上的任何东西。

### 创建元素

1. 调用`React.createElement(...)`
2. 使用 JSX

这里我们深入讨论一下 JSX，在 React17 以前，JSX 会被转换为`React.createElement(...)`调用，这也是使用 JSX 为什么一定要引入 React 的原因。

在 React17 中引入了新的入口函数，不再转换为`React.createElement(...)`调用，但是效果是相同的。

_[关于新的 JSX 转换的博客](https://zh-hans.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)_

## React 组件

不管是类组件还是函数组件，都是接收 props 参数，然后返回一个元素树（element tree）。

只有类组件有实例，但是实例不用我们创建，React 会负责这一部分。

_使用 Refs 保存实例算是一个特例，这仅仅是让我们能够访问其他组件的 DOM_

## 协调

在说协调之前，我们说说 virtual dom，把 vdom 看成一种模式比一种确切的技术要准确得多。

vdom 是用轻量化 js 来表示 UI，相比直接操作真实 dom，使用 vdom 的代价要小得多。通过比较两个 vdom，可以通过算法快速计算出两者的不同，然后最小化的更新真实的 dom。

在 React 中 element 和 fiber 是 virtual dom 中很重要的部分，element tree 是描述了用户界面，fiber 中包含了组件树的附加信息。

现在我们来说说什么是协调。第一次渲染时创建了一个 element tree，当 state 变化、props 变化或者其他需要更新页面的请求到来时，重新生成了一个 element tree，React 需要基于这两者之前的差别高效的更新 UI 以保持同步。准确的来说，协调是一个过程。

协调算法是通用的，React 通过不同的渲染器（React DOM，React Native，React Test）可以在不同的平台渲染 UI。
