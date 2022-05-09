---
title: "DOM事件流"
excerpt: ""
date: 2022-05-07
last_modified_at: 2022-05-07
categories:
  - Javascript
tags:
  - Javascript
---

这篇文章简单介绍一下 DOM 的事件流，事件流主要分为三个阶段，下面是事件流的图解。

<figure>
  <img src="/assets/images/eventflow.svg">
</figure>

- 捕获阶段：从 window 到 target 的父节点
- 目标阶段/处于目标阶段：到达 target
- 冒泡阶段：如果事件类型不支持冒泡，那就没有这个阶段。与捕获阶段的顺序相反直到 window。
- preventDefault()可以阻止默认行为，不阻止传播
- stopPropagation()方法可以阻止事件的进一步传播，不阻止默认行为，也不能阻止绑定在这个目标上的其他相同类型的事件处理函数的执行，如果要阻止需要调用 stopImmediatePropagation()

下面是一个简单的例子

<pre>
....................
. div              .
.  ..............  .
.  . p          .  .
.  ..............  .
....................
</pre>

```javascript
const p = document.querySelector("p");
p.addEventListener(
  "click",
  (event) => {
    console.log("1");
  },
  false
); // false表示在冒泡阶段执行

p.addEventListener(
  "click",
  (event) => {
    console.log("2");
    // 阻止传播，但是绑定在这个元素的同类型事件处理函数仍然会执行
    event.stopPropagation();
  },
  false
);

p.addEventListener(
  "click",
  (event) => {
    console.log("3");
    // 阻止传播，剩下的事件处理函数不会执行
    event.stopImmediatePropagation();
  },
  false
);

p.addEventListener(
  "click",
  (event) => {
    // 这个事件处理函数不会执行
    console.log("4");
  },
  false
);

const div = document.querySelector("div");
div.addEventListener(
  "click",
  (event) => {
    // 这个事件处理函数不会执行，没有冒泡
    console.log("5");
  },
  false
);
```
