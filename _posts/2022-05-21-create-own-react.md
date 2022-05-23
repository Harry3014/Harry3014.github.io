---
title: "从零开始创建React"
excerpt: ""
toc: true
date: 2022-05-21
last_modified_at: 2022-05-21
categories:
  - React
tags:
  - React
---

这篇文章介绍从零开始创建 React。
_React 的真实实现并非如此，我们只是理解 React 设计的哲学_

## 第一步：了解 React Element 的数据结构

我们使用的 JSX，最后转换成 React Element。element 实际是用一个 object 来表示，我们这里只使用其中两个属性，`type`和`props`，实际上它还有更过其他属性，[参考 React 源码](https://github.com/facebook/react/blob/f4cc45ce962adc9f307690e1d5cfa28a288418eb/packages/react/src/ReactElement.js#L111)。例如下面的示例，一段 JSX 实际可以使用纯 object 的形式来表示。

```jsx
<div>
  <p>Are you sure?</p>
  <DangerButton>Yes</DangerButton>
  <Button color="blue">Cancel</Button>
</div>;

const element = {
  type: "div",
  props: {
    children: [
      {
        type: "p",
        props: {
          children: "Are you sure?",
        },
      },
      {
        type: DangerButton,
        props: {
          children: "Yes",
        },
      },
      {
        type: Button,
        props: {
          color: "blue",
          children: "Cancel",
        },
      },
    ],
  },
};
```

## 第二步：`React.createElement`函数

在 React17 之前，JSX 会被转换成`React.createElement(...)`函数调用，这也是为什么之前使用 JSX 一定要引入 React 的原因。

而在 React17 中引入了新的[入口](https://zh-hans.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)，JSX 不再转换成`React.createElement(...)`函数调用，但是我们这里依然讨论这个函数，因为这并不影响问题的本质。
