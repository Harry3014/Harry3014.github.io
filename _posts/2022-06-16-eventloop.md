---
title: "事件循环"
excerpt: ""
date: 2022-06-16
last_modified_at: 2022-06-16
categories:
  - JavaScript
tags:
  - JavaScript
---

我们可能从很多地方都听到过 JavaScript 是单线程语言，单线程意味着同一个时间只能执行一个任务，其他任务必须等到前面任务执行完成后才能执行。

那 JavaScript 是怎么安排异步回调函数的呢？例如异步 Ajax 的响应处理，setTimeout 的回调函数，DOM 绑定的事件监听器等等。

这时候我们就需要来了解事件循环这个模型了。

这个模型中有一个任务队列，当执行栈是空的时候，就拿出队列里的第一个任务开始执行。下面我们以一个实际的例子进行说明。

```javascript
console.log("start");

function printHello() {
  setTimeout(() => console.log("hello"), 5000);
}

printHello();
```

1. 创建全局执行上下文，推入执行栈；
2. 执行`console.log("start")`时创建一个函数执行上下文并推入执行栈中，这个函数完成时，上下文弹出执行栈；
3. 执行`printHello()`，创建一个函数执行上下文并推入执行栈；
4. 执行`setTimeout(...)`时创建一个函数执行上下文并推入执行栈中，这个函数完成时，上下文弹出执行栈；注意这时候有另一个线程专门负责定时器计时；
5. `printHello()`执行完成，它的上下文弹出执行栈；
6. 主程序也执行完成，全局上下文弹出执行栈；
7. 5 秒钟后，负责计时的线程把回调函数放入任务队列中，此时执行栈为空，那么拿出这个任务执行；
8. 执行`console.log("hello")`，完成后弹出上下文，至此整个程序执行完成。

事件循环的概念就是 JavaScript 引擎重复等待任务然后执行任务的这个过程。

这里说的任务也可以称为宏任务，例如运行 script 脚本，setTimeout 回调，ajax 的响应处理，dom 事件处理等都是宏任务。

与宏任务对应的还有一种任务叫微任务。微任务一般由我们的代码创建，例如 promise 的 then/catch/finally，或者 queueMicrotask 函数添加的微任务。

在完成一个宏任务后，会执行所有微任务队列中的所有微任务。然后执行下一个宏任务，或者进行渲染。

看看下面这个示例。

```javascript
setTimout(() => console.log("timeout"), 0);
Promise.resolve("promise").then((value) => console.log(value));
console.log("code");
```

1. 首先输出 code；
2. 然后是 promise，then 方法添加了一个微任务；
3. 然后是 timeout，因为 setTimeout 是另一个宏任务，需要先把微任务队列的所有任务执行完成才会开始下一个宏任务。

事件循环图如下。

<figure>
  <img src="/assets/images/eventLoop-full.svg">
</figure>

如果在执行微任务时添加一个微任务，那么这个微任务会在下一个宏任务前执行，因为要保证微任务队列中的所有任务都执行完成才会开始下一个宏任务。

这个模型有一个缺点，那就是由于同一时间只能处理一个任务，如果这个任务需要很长时间处理，那用户无法跟页面进行交互。
