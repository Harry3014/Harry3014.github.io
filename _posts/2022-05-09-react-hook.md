---
title: "React hooks有魔法？不，它就是数组！！！"
excerpt: ""
date: 2022-05-09
last_modified_at: 2022-05-09
categories:
  - React
tags:
  - React
---

调用 Hook 的时候没有任何特殊的标记，那怎么能记住正确的状态呢？我们来探究一下 Hook 的本质你应该就明白了。

_这篇文章仅仅是 hook 实现原理的简单版本，真正 React 实现肯定要复杂很多_

先来看一个组件

```jsx
function Component() {
  const [firstName, setFirstName] = useState("Rudi");
  const [lastName, setLastName] = useState("Yardley");

  return <button onClick={() => setFirstName("Fred")}>Fred</button>;
}
```

## 初始状态

创建两个空的数组（每个组件实例的 hook 数组是不同的），分别表示`state`和`setter`函数，`cursor`初始值是 0

<figure>
  <img src="/assets/images/react-hook1.png">
</figure>

## 初次渲染

每调用一次，就把`state`的初始值和一个新的`setter`函数加入到各自的数组中，`cursor`加 1

<figure>
  <img src="/assets/images/react-hook2.png">
</figure>

## 再次渲染

每次渲染后`cursor`都重置为 0，然后直接从数组中读取数据

<figure>
  <img src="/assets/images/react-hook3.png">
</figure>

## 调用 setter 函数

调用`setter`本质就是把相同位置的`state`重新修改就好了

<figure>
  <img src="/assets/images/react-hook4.png">
</figure>

下面是`useState`的一个简版实现，仅仅是一种思路

```javascript
// 用一个二维数组来简化state和setter数组
let componentHooks = [];
let currentHookIndex = 0;

function useState(initialState) {
  let pair = componentHooks[currentHookIndex];

  // 已经存在数据了，表示非初次渲染，直接返回，并且移动指针准备等待下一次调用
  if (pair) {
    currentHookIndex++;
    return pair;
  }

  // 初次渲染，state设置为初始值，创建新的setter函数
  pair = [initialState, setState];

  function setState(nextState) {
    // nextState可能是函数，这里不做详细的实现
    pair[0] = nextState;
    // 后续还有一些操作，不做详细实现
  }

  componentHooks[currentHookIndex] = pair;
  currentHookIndex++;

  return pair;
}
```

现在我们来看看 Hook 的[使用规则](https://zh-hans.reactjs.org/docs/hooks-rules.html)

- 只能在最顶层调用 Hook，不能在循环，条件或者嵌套函数中调佣
- 只能在函数组件或者自定义 Hook 中调用 Hook

如果理解了上面的内容，那么你应该能理解第一条规则。如果还是有点模糊，继续往下看。

看看下面错误示范，在条件中调用 hook。

```jsx
let firstRender = true;

function Component() {
  let initName;
  if (firstRender) {
    [initName] = useState("Rudi");
    firstRender = false;
  }

  const [firstName, setFirstName] = useState("Rudi");
  const [lastName, setLastName] = useState("Yardley");

  return <button onClick={() => setFirstName("Fred")}>Fred</button>;
}
```

第一次渲染，看起来都很正常

<figure>
  <img src="/assets/images/react-hook5.png">
</figure>

第二次渲染，`firstName`获取到的实际上是`initName`的值，`lastName`获取到的是`firstName`的值，都是"Rudi"，这全乱套啦。

<figure>
  <img src="/assets/images/react-hook6.png">
</figure>

如果严格遵守第一条规则，那么 React 就能通过调用 Hook 的顺序来获取多个 Hook 的正确状态。
