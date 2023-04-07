---
title: "前端基础概念及常见问题解答"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-04-04
last_modified_at: 2023-04-04
categories:
  - Frontend
tags:
  - Frontend
---

## javascript 基础概念

### 作用域

大致可以分为全局作用域，函数作用域，块作用域。

作用域通过环境记录 Environment Record 实现，环境记录中记录了变量、函数等标识符，还有一个指针[[OuterEnv]]指向外部的环境记录，直到全局环境的外部环境记录为 null，这样就形成了一个链条。

当使用变量和函数时，先在当前自己的环境记录中寻找，如果没有找到则在外部的环境中寻找，一直到全局环境。

如果未找到变量则会抛出引用错误。（注意这里与变量赋值区分）

下面是一个简单的示例

<figure>
  <img src="/assets/images/lexical-environment-simple-lookup.svg">
</figure>

- name 变量存在于 say 函数的环境记录中
- pharse 变量存在全局环境记录中，由于 say 函数的环境记录引用了外部的全局环境记录，所以在 say 函数中可以访问 pharse 变量

下面是一个创建内部函数的示例

<figure>
  <img src="/assets/images/closure-makecounter-nested-call.svg" />
</figure>

创建的内部函数的环境记录的外部环境是 makeCounter 函数的环境记录，所以能够访问 count 变量。

每次调用函数都会创建新的环境记录，所以即使多次调用 makeCounter，他们内部的值也不会互相影响。

```javascript
function makeCounter() {
  let privateCount = 0;

  function changeBy(val) {
    privateCount += value;
  }

  return {
    increment: function () {
      changeBy(1);
    },
    decrement: function () {
      changeBy(-1);
    },
    value: function () {
      return privateCount;
    },
  };
}

const Counter1 = makeCounter();
const Counter2 = makeCounter();

console.log(Counter1.value()); /* logs 0 */
Counter1.increment();
Counter1.increment();
console.log(Counter1.value()); /* logs 2 */
Counter1.decrement();
console.log(Counter1.value()); /* logs 1 */

console.log(Counter2.value()); /* logs 0 */
```

### 闭包

闭包是一个函数和它引用的外部环境记录的组合，闭包使得在函数内部能够访问外部函数的作用域。

按照这种定义，每一个函数都是闭包的，因为函数至少会引用全局环境。

闭包非常有用，因为它可以访问外部函数的作用域，还可以像上面的 makeCounter 示例一样模拟私有方法。

虽然闭包有诸多优点，但是也有缺点。按照常理，当调用函数完成后，函数内部的变量应该执行垃圾回收，但是由于闭包的存在，内部函数仍然引用着外部环境，所以无法回收。但是 js 引擎可能会做出一些优化，如果内部函数中没有使用外部函数的变量，那么还是可能会回收的。

例如下面这个例子，内部函数未使用 name 变量，那么其实可以优化。

```javascript
function makePrinter() {
  const name = "harry";
  return function () {
    console.log("jack");
  };
}

const printer = makePrinter();
printer();
```

如果确定不再使用闭包函数，可以将其设置为 null，使其内存可以被回收，否则可能造成内存泄漏。

### var&函数声明提升&let，const 暂时性死区

使用 var 声明变量和函数声明，都会在创建环境记录时一并创建，这叫变量提升和函数声明提升，只是 var 变量的值为`undefined`。

而 let 和 const 声明变量，也会在创建环境记录时一并创建，但是在未运行初始化之前，这些变量都是不可访问的，这一块区域叫暂时性死区。

来看看下面的例子，函数声明提升使得可以在声明之前调用，var 变量声明提升使得可以提前访问，但是值为`undefined`。

const 变量在未初始化之前都处于暂时性死区，不能访问，否则抛出引用错误。

```javascript
print();

function print() {
  console.log(firstName); // undefined
  console.log(lastName); // ReferenceError
}

var firstName = "Jack";
const lastName = "Ma";
```

### 对象&原型

大致可以通过下面几种方式创建对象。

- 对象字面量
- 使用构造函数
- Object.create(proto)，这种不是很常用

所有的对象都至少继承一个对象，继承的对象叫做原型 prototype，对象的私有属性\_\_proto\_\_指向原型。

在创建函数时会创建原型对象，prototype 属性指向原型，原型的 constructor 属性指向函数。

原型还有它自己的原型，直到 Object 函数原型的原型为 null，这样就构成了原型链。

```javascript
Object.prototype !== null;
Object.prototype.__proto__ === null;
```

当查找对象属性时，会从对象本身开始，如果没有找到就沿着原型链往上查找。

举一个例子。

```javascript
function User(id, name) {
  this.id = id;
  this.name = name;
}

User.prototype.sayHello = function () {
  console.log(`My name is ${this.name}`);
};

let user = new User(1, "michael");
user.sayHello(); // My name is michael
```

user 实例的结构如下。

```javascript
user = {
  id: 1,
  name: "michael",
  __proto__: User.prototype,
};
```

在上面的例子中，user 实例的原型链是 user -> User.prototype -> Object.prototype -> null。

user 继承了 User 函数的原型，所以可以找到 sayHello 属性进行调用。

User 函数本身也是一个对象，它的原型链是 User -> Function.prototype -> Object.prototype -> null。

User 函数继承了 Function 函数的原型，可以 User 函数可以调用 bind，apply 等函数。

### promise

```javascript
const promise = new Promise(executor);
function executor(resolve, reject) {
  // 通常是一些异步操作
}
```

通过 Promise 构造函数创建 promise 有以下需要注意的点：

- executor 在 promise 创建的期间被执行
- resolve 和 reject 在创建 promise 时同时被创建，并且绑定该 promise
- executor 中通常用于执行一些异步操作
- 通过调用 resolve，reject 反应异步操作的结果，否则 promise 的状态一直是 pending
- 如果在 executor 中发生了错误，那么 promise 状态变为 rejected
- executor 的返回值会被忽略

还有一种特殊情况也要注意：调用 resolve，reject 传入另一个 promise

首先来看 resolve 传入 promise

```javascript
const promise = new Promise((resolve, reject) => {
  resolve(
    new Promise((resolve, reject) => {
      setTimeout(() => {
        reject("uu");
      }, 3000);
    })
  );
});

promise.then(
  (val) => console.log(val),
  (reason) => console.log("rejected:" + reason)
);
// 3秒后输出rejected:uu
// 虽然调用了promise绑定的resolve函数，但是resolve传入了一个新的promise，它的最终状态是rejected
```

reject 传入 promise 情况则有所不同

```javascript
const promise = new Promise((resolve, reject) => {
  reject(
    new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve("uu");
      }, 3000);
    })
  );
});

promise.then(
  (val) => console.log(val),
  (reason) => console.log("rejected:" + reason)
);
// 立即输出rejected:[object Promise]
// promise最终的状态仍然是rejected，并且拒绝原因是传入的那个promise
```

所以我们应该避免 reject 传入 promise。

### promise 链式调用

在一个异步操作完成后才进行下一个操作这是很常见的一个需求，可以通过 promise 链式调用来实现。

链式调用通过下面这三个方法实现，它们都返回一个新的 promise。

```
Promise.prototype.then(onFulfilled, onRejected)
Promise.prototype.catch(onRejected)
Promise.prototype.finally(onFinally)
```

关于链式调用，有以下需要注意的地方：

- then 返回的 promise 的最终状态由 onFulfilled 或者 onRejected 的返回值决定（两者中只有一个被调用）

  - 抛出错误：状态变为 rejected，原因是抛出的错误。注意：下面这种异步抛出的错误无法处理，已经 resolve 了之后抛出的错误也无法处理。

    ```javascript
    new Promise((resolve, reject) => {
      setTimeout(() => {
        throw "oops";
      });
      // resolve(1);
      // throw "oops" 一样无法处理
    }).then(
      (result) => {
        console.log("resolved:" + result);
      },
      (reason) => {
        console.log("rejected:" + reason);
      }
    );
    ```

  - 返回新的 promise，根据返回 promise 的状态不同，then 返回的 promise 也有不同

    - 新的 promise 是 fulfilled，那么也变为 fulfilled，结果也是一样的
    - 新的 promise 是 rejected，那么也变为 rejected，拒绝原因也是一样的
    - 新的 promise 是 pending，那么等到它敲定状态，then 返回的 promise 也敲定一样的状态

  - 返回其他值和 undefined，那么变为 fulfilled，结果是返回值或者 undefined

- 如果 then 的两个状态回调函数都不是有效的函数，那么 then 返回的 promise 的状态仍然跟随调用它的 promise 的状态，只是没有回调。

- catch 类似于 then(undefined, onRejected)

- finally 的 onFinally，在调用 promise 敲定后，无论什么状态都会调用，所以它没有参数。

- finally 返回的 promise 的最终状态与 then 不同，then 是根据回调函数的返回值，而 finally 返回的 promise 跟随调用它的 promise。

  ```javascript
  Promise.resolve(2).then(
    () => {},
    () => {}
  ); // 返回的promise最终状态为resolved，结果为undefined，因为回调函数没有返回值
  Promise.resolve(2).finally(() => {}); // 返回的promise最终状态为resolved，结果为2
  
  Promise.reject(3).then(
    () => {},
    () => {}
  ); // 返回的promise最终状态为resolved，结果为undefined，即使原调用then的promise是rejected
  Promise.reject(3).finally(() => {}); // 返回的promise最终状态为rejected，拒绝原因为3
  ```

### async & await

async 函数返回一个 promise，promise 最终的状态必然是以下两种情况之一：

- 由 async 函数的返回值 resolve
- async 抛出异常 reject，抛出异常的原因可能也有两种：
  - await 等待的 promise 被拒绝
  - 函数本身的代码逻辑抛出异常

下面来看一个简单的例子。

```javascript
function resolveAfter2Seconds() {
  return new Promise((resolve) => {
    console.log("starting slow promise");
    setTimeout(() => {
      resolve("slow result");
      console.log("slow promise is done");
    }, 2000);
  });
}

function resolveAfter1Second() {
  return new Promise((_, reject) => {
    console.log("starting fast promise");
    setTimeout(() => {
      reject("fast error");
      console.log("fast promise is rejected");
    }, 1000);
  });
}

async function asyncCall() {
  console.log("async function start");

  try {
    const slow = await resolveAfter2Seconds();
    console.log(slow);
  } catch (e) {
    console.log(e);
  }

  try {
    await resolveAfter1Second();
  } catch (e) {
    console.log(e);
  }
}

const promise = asyncCall();

console.log("after call async function");
```

console 中依次输出：

```
// 调用async函数
async function start
// slow promise开始
starting slow promise
// 由于等待的slow promise没有敲定，所以要暂停async函数让出控制权
after call async function
// 2秒后slow promise敲定
slow promise is done
// 变量slow被赋值为slow promise敲定的值
slow result
// fast promise开始
starting fast promise
// 等待的fase promise没有敲定，也要暂停让出控制权
// 1秒后fast promise被拒绝
fast promise is rejected
// 被拒绝后把拒绝原因作为错误抛出
// 捕获到这个错误
fast error
```

到这里 async 函数结束，返回了 undefined，所以 promise 的状态变为 fulfilled，promise 的结果为 async 函数的返回值即 undefined。

如果在 async 函数中没有捕获处理错误，那么 promise 的状态会变为 rejected，结果为 reject 的原因。

## 常见问题

### 事件循环

为了协调事件，交互，脚本，渲染，网络请求等等，浏览器或者其他用户代理（例如邮件阅读器）使用事件循环模型来管理协调这些内容。

这些事件可以理解为任务，当事件发生时，就将任务放进相应的任务队列，不止一个任务队列，微任务队列是一个特殊的队列，不属于任务队列。

例如script脚本，事件分发，回调，定时器，messageChannel发送消息等等都属于任务。

那么事件循环模型是如何运作的呢？它就相当于是一个持续的循环结构。

```javascript
while(true) {
  // 从任务队列中取出一个任务执行
  queue = getNextQueue();
  task = queue.pop();
  excute(task);
  
  // 执行目前微任务队列中的所有微任务，queueMicroTask，promise的状态回调
  while(microTaskQueue.hasTasks()) {
    doMicroTask();
  }
  
  // 如果需要渲染
  if(isRepaintTime()) {
    // 先执行动画相关的任务requestAnimationFrame
    animationTasks = animationQueue.copyTasks();
    for (task in animationTasks) {
      doAnimationTask(task);
    }
    
    repaint();
  }
}
```

