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

_这里 children 固定为数组以及新增一种 text 类型，都是仅仅为了实现方便，React 源码并非如此_

_React.createElement 的[源码实现](https://github.com/facebook/react/blob/f4cc45ce962adc9f307690e1d5cfa28a288418eb/packages/react/src/ReactElement.js#L312)_

```js
function createElement(type, config, ...children) {
  return {
    type,
    props: {
      ...config,
      children: children.map((child) => {
        if (typeof child === "string") {
          return createTextElement(child);
        } else {
          return child;
        }
      }),
    },
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
}
```

## 第三步：`ReactDOM.render`函数

现在我们讨论如何把 element 渲染出来。根据 element 的类型来创建不同类型的 DOM 元素，然后添加到 DOM 树中即可。

_这里实现的是 ReactDOM.render 方法，此方法已在 React18 中被 createRoot 函数取代_

```js
function render(element, container) {
  let dom = null;
  if (element.type === "TEXT_ELEMENT") {
    dom = document.createTextNode("");
  } else {
    dom = document.createElement(element.type);
  }

  Object.keys(element.props)
    .filter((key) => key !== "children")
    .forEach((key) => (dom[key] = element.props[key]));

  element.props.children.forEach((child) => render(child, dom));

  container.appendChild(dom);
}
```

在[codesanbox](https://codesandbox.io/s/fakereact-ls247r?file=/src/index.js)中试试吧！

## 第四步：调度

如果把一个结构很复杂的 element 没有中断的一次性渲染出来，那么主线程将会被一直占用，一些高优先级的事务都会被阻塞。

那么我们就应该把这部分工作分成一个个小的单元，让我们能够控制什么时候开始和中止。

这里我们使用`requestIdleCallback`来实现。检查是否存在需要执行的任务，是否空闲，如果满足条件，那么执行这个任务并返回下一个任务，重复这个过程。

_React 没有使用 requestIdleCallback，而是自己实现了 scheduler_

```js
let nextUnitOfWork = null;

requestIdleCallback(workLoop);

function workLoop(deadline) {
  let idle = true;
  while (nextUnitOfWork && idle) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    idle = deadline.timeRemaining() > 1;
  }
  requestIdleCallback(workLoop);
}

function performUnitOfWork(unitOfWork) {
  // TODO
  return null;
}
```

## 第五步：了解 Fiber 结构

这是一段 jsx，下面的图描述了对应的 Fiber 树。

`fiber.child`表示第一个 child，`fiber.sibling`表示同一层级的下一个 fiber，`fiber.parent`表示父节点。这样类似链表结构很方便进行操作。

```html
<div>
  <h1>
    <p />
    <a />
  </h1>
  <h2 />
</div>
```

<figure style="background: #263238">
  <img src="/assets/images/fibertree.png">
</figure>

上一步我们还没有实现`performUnitOfWork`，这个函数做三件事。

1. 把 React 元素添加到 DOM 中
2. 为 children 元素生成 fiber
3. 选择下一个要执行的任务

```js
function performUnitOfWork(fiber) {
  // 添加DOM
  if (!fiber.dom) {
    fiber.dom = createDOM(fiber);
  }

  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom);
  }

  // 生成子元素的fiber，指定child，sibling，parent
  const elements = fiber.props.children;

  let isFirstChild = true;
  let leftSibling = null;
  for (const element of elements) {
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    };
    if (isFirstChild) {
      fiber.child = newFiber;
      isFirstChild = false;
    } else {
      leftSibling.sibling = newFiber;
    }
    leftSibling = newFiber;
  }

  /**
   * 选择下一个要执行的fiber
   * 如果有child，直接返回
   * 否则检查是否有sibling
   * 否则返回parent，检查parent是否有sibling
   */
  if (fiber.child) {
    return fiber.child;
  }

  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }

  return nextFiber;
}
```

## 第六步：渲染和提交阶段

上一步的`performUnitOfWork`中我们处理一个 fiber 的时候立刻添加了 DOM，我们忽略了一个问题就是我们是利用浏览器的空闲时段来处理，是可能随时被中断的，那么呈现出来的可能是不完整的 UI，这不是我们所期望的。

那我们就保存 fiber root，等到没有任务了，然后再提交 DOM。

```js
let wipRoot = null;

function commitRoot() {
  commitWork(wipRoot.child);
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }
  fiber.parent.dom.appendChild(fiber.dom);
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```

## 第七步：协调

现在我们只讨论了添加 DOM 的情况，那么更新和删除呢？

我们需要保存当前的 fiber 树，然后跟新的 fiber 树进行比较就可得出需要添加更新删除的信息。

比较过程如下：

1. 如果有相同的 type，那么就是更新 DOM
2. 如果 type 不同，并且有元素，那么需要给这个元素创建 DOM
3. 如果 type 不同，而且存在旧的 fiber，那么需要删除 DOM
