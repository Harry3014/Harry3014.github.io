---
title: "置顶🔝"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-06-10
last_modified_at: 2023-06-10
categories:
  - Frontend
tags:
  - Frontend
---

## Javascript

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

User 函数继承了 Function 函数的原型，所以 User 函数可以调用 bind，apply 等函数。

### typeof & instanceof

typeof 运算符返回字符串表示操作数的类型。以下情况使用 typeof 值得注意。

```javascript
typeof undefinedValue === "undefined";
typeof null === "object";
typeof NaN === "number";
typeof new String("xx") === "object";
typeof undeclaredVar === "undefined"; // 未声明变量不会报错，但是如果是let，const变量在暂时性死区会报错
```

instanceof 运算符用于检测函数的原型是否在对象的原型链上，例如`object instanceof constructor`就是检查`constructor.prototype`在不在 object 的原型链上。

左值可以为任何类型，但是只有对象才会检测原型链。

右值只能为可调用的函数。

一下情况使用 instanceof 值得注意。

在多个窗口之间交互，不同的窗口拥有不同的内置构造函数，例如 iframe 中的 Array 构造函数与 top 中的不一样，所以使用 instanceof 并不是保险的。例如判断是否是数组应该使用：

```javascript
Array.isArray(testObj);
Object.prototype.toString.call(testObj) === "[object Array]";
```

### this

this 的值是在代码运行时绑定，取决于执行上下文。

**全局上下文**

全局执行环境下 this 指向全局对象，例如浏览器指向 window。

**非箭头函数上下文**

函数中的 this 取决于如何调用它。

- 在全局上下文中直接调用

  ```javascript
  function func() {
    return this;
  }

  func(); // 非严格模式指向全局对象，严格模式下为undefined
  ```

- 作为对象方法调用，this 指向调用它的对象

  ```javascript
  let obj = {
    name: "test",
    print: function () {
      return console.log(this.name);
    },
  };

  obj.print();
  ```

- 作为构造函数，this 指向一个新的对象

  ```javascript
  function Person(firstName, lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }
  ```

  这里顺便说一下使用 new 调用构造函数生成一个对象的过程

  - 创建一个空的简单对象`{}`
  - 对象的`__proto__`指向构造函数的原型对象
  - 将新创建的对象作为 this
  - 执行构造函数内容
  - 如果没有返回值，默认返回这个对象

**箭头函数上下文**

箭头函数没有自己的 this，它继承于作用域链上的上一层中 this。

顺便说一下，箭头函数无法作为构造函数，

**类上下文**

类上下文中与函数类似。

**apply&call 指定 this 调用**

函数的 apply 和 call 方法可以指定 this 进行调用，如果指定的 this 参数不是对象会尝试转换为对象，如果是 null 或 undefined，在非严格模式下是全局对象。

**bind**

调用 bind 返回一个新的方法，这个方法中的 this 被永久设置为传入的参数。

```javascript
function f() {
  return this.a;
}

var g = f.bind({ a: "azerty" });
console.log(g()); // azerty

var h = g.bind({ a: "yoo" }); // bind 只生效一次！
console.log(h()); // azerty
```

**事件处理函数**

this 指向事件绑定的元素，即`this === event.currentTarget`，`event.target`指向触发事件的元素。

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
- 通过调用 resolve，reject 反映异步操作的结果，否则 promise 的状态一直是 pending
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

### 生成器

生成器函数，返回 Generator 对象。

```javascript
function* gen() {
  try {
    let id = 0;
    while (true) {
      const step = yield ++id;
      if (typeof step === "number") {
        yield (id += step);
      }
    }
  } catch (error) {
    console.log(error); // test error
  }
}

const generator = gen();

console.log(generator.next()); // {value: 1, done: false}
// 向生成器传入值
console.log(generator.next(5)); // {value: 6, done; false}
console.log(generator.next()); // {value: 7, done: false}
console.log(generator.next()); // {value: 8, done: false}
// 向生成器抛出错误，这里因为已经到迭代序列的末尾，所以done为true
console.log(generator.throw("test error")); // {value: undefined, done: true}
console.log(generator.next()); // {value: undefined, done: true}
```

```javascript
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}

const generator = gen();

console.log(generator.next()); // {value: 1, done: false}
// 结束生成器
console.log(generator.return("foo")); // {value: 'foo', done; true}
console.log(generator.next()); // {value: undefined, done: true}
```

### localStorage&sessionStorage

它们用于同一个源下的本地存储，并且提供相同的方法。

- setItem(string, string)，如果提供非 string，会转成 string
- getItem
- removeItem
- clear

localStorage 可以长期保存，sessionStorage 仅仅在会话期间保存，sessionStorage 同一个 URL 打开不同标签也会创建各自的 sessionStorage。

### WeakMap 与 Map 的区别

为什么会引入 WeakMap，因为 Map 的键和值一直被引用，可能会导致内存泄漏，除非手动删除，而 WeakMap 的键是弱引用的，是可以被回收的。

WeakMap 的键一定是对象，它的键不可枚举，所以无法迭代。

[参考 MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Keyed_collections)

## 常见问题

### 事件循环

[跳转 YouTube](https://youtu.be/u1kqx6AenYw)

为了协调事件，交互，脚本，渲染，网络请求等等，浏览器或者其他用户代理（例如邮件阅读器）使用事件循环模型来管理协调这些内容。

它是 JavaScript 作为一门单线程编程语言实现非阻塞处理 I / O 等事件的方式。

这些事件可以理解为任务，当事件发生时，就将任务放进相应的任务队列，不止一个任务队列，微任务队列是一个特殊的队列，不属于任务队列。

例如 script 脚本，事件分发，回调，定时器，messageChannel 发送消息等等都属于任务。

promise callback，MutationObserver callback 属于微任务，也可以通过 queueMicroTask API 添加微任务。

那么事件循环模型是如何运作的呢？它就是一个持续运行的循环结构。

<figure>
  <img src="/assets/images/eventLoop-full.svg">
</figure>

```javascript
while (true) {
  // 从任务队列中取出一个任务执行
  queue = getNextQueue();
  task = queue.pop();
  excute(task);

  // 执行目前微任务队列中的所有微任务，queueMicroTask，promise的状态回调
  while (microTaskQueue.hasTasks()) {
    doMicroTask();
  }

  // 如果需要渲染
  if (isRepaintTime()) {
    // 先执行动画相关的任务requestAnimationFrame
    animationTasks = animationQueue.copyTasks();
    for (task in animationTasks) {
      doAnimationTask(task);
    }

    repaint();
  }
}
```

### 浏览器渲染页面

DNS 查询得到 IP 地址

三次握手建立 TCP 连接

- 浏览器发送 SYN，假设序列号是 x
- 服务器收到请求，发送 SYN，序列号是 y，还要发送 ACK：x+1
- 浏览器收到 ACK，发送 ACK：y+1。需要这一次握手的原因：浏览器发送的建立请求因为网络延迟很久才到服务器，服务器无法知晓这个请求是否过期，如果这样就建立了连接，那么是资源的浪费，因为这个请求可能已经过期了。而浏览器是知道这个请求是否已经过期，假设已经过期，那么就不会回复 ACK 建立连接。

如果使用 https 协议，还会进行 TLS 协商以建立安全的连接

发起 HTTP 请求，通常是获取 HTML 文件

如果能够成功后去，浏览器会解析 HTML

- 构建 DOM 树（文档对象模型），在解析 HTML 标签的过程中，没有 async 或 defer 的 script 会阻塞解析 HTML，但是一些高优先级的资源会在解析 HTML 前就发起请求获取，例如 css，js 文件等，这样能减少阻塞。

  顺带提一下 async 和 defer 属性，async 会并行请求，defer 会延迟到解析完成后请求，在 DOMContentLoaded 事件触发前。

  load 事件是在所有资源都加载完成后才会触发，包括图片等。

- 构建 CSSOM，构建 CSSOM 的速度是非常快的

- 在解析 HTML 的过程中还会执行 script

解析完成后就可以进行渲染

- 将 DOM 和 CSSOM 结合成 Render 树，根据 CSS 级联规则确定节点的计算样式
- 确定节点的尺寸和位置，第一次称为布局（layout），再次计算称为回流（reflow）
- 绘制（paint）节点到屏幕，例如文本，颜色，边框等等，再次绘制称为重绘（repaint）
- 如果有相互重叠时，还需要合成（compositing），确保以正确的顺序显示

渲染完成后不代表一定能立马进行交互，可能还在执行脚本，此时无法进行交互

### 浮点数运算精度缺失

原因：十进制的二进制表示形式可能不精确，例如 0.1，只有 1 除以 2<sup>n</sup>的小数才能被精确表示。

解决方法：

- 绝对值与`Number.EPSILON`进行比较
- 使用一些库，例如 decimal.js

### 建立 tcp 三次握手

[跳转 RFC](https://www.rfc-editor.org/rfc/rfc9293#name-establishing-a-connection)

<pre>    TCP Peer A                                           TCP Peer B

1.  SYN-SENT    --&gt; &lt;SEQ=100&gt;&lt;CTL=SYN&gt;               --&gt; SYN-RECEIVED

2.  ESTABLISHED &lt;-- &lt;SEQ=300&gt;&lt;ACK=101&gt;&lt;CTL=SYN,ACK&gt;  &lt;-- SYN-RECEIVED

3.  ESTABLISHED --&gt; &lt;SEQ=101&gt;&lt;ACK=301&gt;&lt;CTL=ACK&gt;       --&gt; ESTABLISHED
</pre>

### 关闭 tcp 连接四次挥手

[跳转 RFC](https://www.rfc-editor.org/rfc/rfc9293#name-closing-a-connection)

<pre>    TCP Peer A                                           TCP Peer B

1.  (Close)
    FIN-WAIT-1  --&gt; &lt;SEQ=100&gt;&lt;ACK=300&gt;&lt;CTL=FIN,ACK&gt;  --&gt; CLOSE-WAIT

2.  FIN-WAIT-2  &lt;-- &lt;SEQ=300&gt;&lt;ACK=101&gt;&lt;CTL=ACK&gt;      &lt;-- CLOSE-WAIT

3.                                                       (Close)
    TIME-WAIT   &lt;-- &lt;SEQ=300&gt;&lt;ACK=101&gt;&lt;CTL=FIN,ACK&gt;  &lt;-- LAST-ACK

4.  TIME-WAIT   --&gt; &lt;SEQ=101&gt;&lt;ACK=301&gt;&lt;CTL=ACK&gt;      --&gt; CLOSED

5.  (2 MSL)
    CLOSED
</pre>

### TLS 握手

<figure>
  <img src="/assets/images/tls-handshake.png">
</figure>

1. 客户端发送 ClientHello 消息，包括支持的 TLS 版本，密码套件（包括加密算法）等信息
2. 服务器收到消息后，判断是否支持相关版本和密码套件，如果支持，那么可以发回已决定的 TLS 版本，密码套件，证书等信息，证书包含颁发机构，公钥等信息
3. 客户端验证证书是否有效，所有验证通过后，客户端使用随机数生成一个会话密钥，并使用公钥加密，然后发回到服务器，只有服务器的私钥才能解密得到会话密钥。后续使用这个会话密钥加密数据。

## HTTP

首先回顾一些基本的概念。

URL 的组成部分

<figure>
  <img src="/assets/images/URI_syntax_diagram.svg.png" />
</figure>

协议://主机:端口/路径?查询#片段

data URL

```
data:[mediatype][;base64],data
```

MIME 类型 type/subtype

描述文档文件、字节流的格式和性质，常见的 text/plain，application/octet-stream

### 跨源资源共享 cors

同源：协议，主机，端口都相同。

**预检请求**

某些类型的跨域请求在发送前需要先发送一个预检请求，服务器处理预检请求告知浏览器是否允许发送真正的跨域请求。

这些需要发送预检请求的跨域请求是：

- 方法限制：不是 get，head，post
- 头部限制：包含了一些对于 cors 来说不安全的头部
- Content-Type 限制
- 还有一些其他的限制，这里不一一列举了

**请求头部字段**

- Origin

  请求源

- Access-Control-Request-Method

  告知服务器真正跨域请求使用的方法，用于预检请求

- Access-Control-Request-Headers

  告知服务器真正跨域请求使用的头部，逗号隔开，用于预检请求

**响应头部字段**

- Access-Control-Allow-Origin

  指定该资源允许被哪些源跨域使用

  - \* 允许任意来源
  - 指定一个来源，如果指定了来源，必须在 Vary 头部中添加 Origin 头部用于表明不同 Origin 会有不同的此头部字段值

  ```http
  Access-Control-Allow-Origin: https://developer.mozilla.org
  Vary: Origin
  ```

- Access-Control-Allow-Methods

  告知浏览器跨域请求允许使用的方法，用于响应预检请求，逗号隔开

- Access-Control-Allow-Headers

  告知浏览器跨域请求允许使用的头部，用于响应预检请求，逗号隔开

- Access-Control-Max-Age

  指定预检请求的响应能够缓存多久，单位秒，默认 5 秒，浏览器有一个最大限制

除了以上常用的响应头部字段，还有一些其他的用于 cors 响应的头部字段，但是不常用。

### http 缓存

http 旨在尽可能多缓存响应，只要满足 http 规定的缓存条件就能缓存响应，这些条件参考<a href="https://www.rfc-editor.org/rfc/rfc9111.html#name-storing-responses-in-caches" target="_blank">RFC9111</a>。

目前使用比较广泛的缓存控制策略是通过 Cache-Control 头，但是不使用某些特定头部也是可以被缓存的，只要响应满足上面说的 RFC9111 规定的条件。

**重用缓存**

在发起请求时重用缓存同样也需要满足一些条件，这些条件参考<a href="https://www.rfc-editor.org/rfc/rfc9111.html#name-constructing-responses-from" target="_blank">RFC9111</a>。

我们比较熟知的可以重用缓存的情况可能是：

- URI 要匹配
- 缓存仍然是新鲜的
- 缓存过期，但是经过重新与服务器验证可以使用过期的缓存
- 如果响应含有 Vary 头，那么 Vary 指定的头匹配才能使用缓存
- 如果响应含有 no-cache 指令，必须经过验证才能使用缓存

其实还存在其他可以重用缓存的情况，例如客户端在 Cache-Control 中使用 max-stale 指令，表明可以接受过期时间不超过此设定的缓存。

**检查新鲜度**

检查缓存是否新鲜的方式是 response_is_fresh = (freshness_lifetime > current_age)。

计算 freshness_lifetime 的方式按照下面的优先级：

- 如果是共享缓存：Cache-Control 的 s-maxage 指令
- Cache-Control 的 max-age 指令
- Expires 头 减掉 Date 头
- 以上都没有就属于启发式缓存，使用 Date 减掉 Last-Modified，然后取 10%

计算 current_age 稍微有些复杂，可以通俗的理解成缓存的当前寿命。

**重新验证**

虽然缓存中可能存在 URI 匹配的缓存，但是由于各种原因无法重用缓存，那么可以发起重新验证请求。

如果响应中包含 ETag 头，在重新验证时使用 If-None-Match，如果服务器检查没有修改，可以返回 304Not Modified。

如果响应中包含 Last-Modified 头，在重新验证时使用 If-Modified-Since，没有修改返回 304。

**Cache-Control 常见指令**

可缓存性：public：共享缓存允许被任何对象缓存，private：只允许单个用户缓存，no-cache：仍然可以缓存，只是重用缓存前需要验证，no-store：不缓存

到期：s-maxage：仅仅能指定共享缓存有效期，max-age：缓存有效期，max-stale：客户端愿意使用过期缓存，最大期限

**缓存模式**

最适合缓存的资源是静态资源，如果资源经常变动，建议在资源改动后修改 URL，例如添加 hash 到文件名，或者添加版本号到查询。

### http cookie

可以在响应中使用 Set-Cookie 头设置 cookie，再次向同一服务器发送请求时会附带 cookie 信息在 Cookie 头中。

**生命周期**可以通过两种方式定义：

- 会话期，在当前会话结束后就删除，不包含任何有效期指令
  ```http
  Set-Cookie: sessionId=38afes7a8
  ```
- 持久型，可以通过 Max-Age 指定有效时长（优先级更高，小于等于 0 时立即失效），通过 Expires 指令指定失效时间
  ```http
  Set-Cookie: id=a3fWa; Max-Age=2592000
  Set-Cookie: id=a3fWa; Expires=Wed, 21 Oct 2015 07:28:00 GMT
  ```

**限制使用 cookie**

- Secure 指令，仅在使用 https 时才发送 cookie，本地主机除外（localhost）

- HttpOnly 指令，无法使用 document.cookie 读取 cookie

```http
Set-Cookie: id=a3fWa; Max-Age=2592000; Secure; HttpOnly
```

**定义哪些地方可以发送 cookie**

- Domain，指定 cookie 可以送达的主机，如果不指定，那么默认就是当前主机（并且不包含子域名），如果指定主机，那么包含子域名。

```http
Set-Cookie: sessionId=e8bb43229de9; Domain=example.com
```

- Path，指定路径

```http
Set-Cookie: sessionId=e8bb43229de9; Path=/docs
```

- SameSite，值为 Strict，Lax，None

可以通过 document.cookie 读取（前提是没有设置 HttpOnly 指令）和写入 cookie。

读取 document.cookie 的值得到 cookie1=value1;cookie2=values;...

可以通过设置 document.cookie=encodeURIComponent(name) + '=' + encodeURIComponent(value)写入 cookie，最好调用 encodeURIComponent 保持有效格式，如果需要设置指令，以分号隔开。

### http 常见状态码

- 101 switching protocol：服务器根据客户端提起的协议升级请求，正在升级协议，例如创建 websocket 时会使用到

  ```http
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  ```

- 200 ok

- 304 not modified，服务器告知缓存没有被修改，因此客户端仍然可以使用此缓存

- 400 bad request 客户端错误的请求

- 403 forbidden 客户端没有访问内容的权限

- 404 not found 服务器找不到相关内容

- 405 method not allow 服务器禁止使用该请求的方法

- 429 too many requests，一定时间内超出了频率限制，在响应中可设置 Retry-After 头部告知需要等待多少秒，ChatGPT 调用 Sentry 接口时见过。

- 500 internal server error 服务器内部错误

- 504 Gateway timeout 扮演网关或者代理的服务器无法在规定的时间内获得想要的响应

### http/1.x 连接管理

[跳转 MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Connection_management_in_HTTP_1.x)

<figure>
  <img src="/assets/images/http1_x_connections.png">
</figure>

http/1.x 的连接管理模型依次出现了：短连接 short-lives，持久连接 persistent，流水线 pipelining。

**短连接**
每个请求完成后就关闭了 tcp 连接，新的请求发出之前都需要再次建立 tcp 连接。

缺点：建立新的 tcp 太耗费资源。

**持久连接**

持久连接允许让 tcp 连接保留一段时间。客户端和服务器可以设置`Connection： keep-alive`通用头部来表明是否支持长连接，还可以通过`Keep-Alive`通用头部设置超时时长和最大请求数。

缺点：tcp 一直保持连接状态也是占用资源的。

**流水线**
流水线建立在持久连接的基础上。

默认情况下，新的请求只有在当前请求收到响应过后才会被发出。而流水线则可以发出连续的请求，而不用等待应答返回。

但是由于 http/1.x 限制，服务器只能按照请求的顺序进行响应，假设某个请求不能及时响应，那么后面的请求也无法响应，这就是队头阻塞问题。http/2 实现了多路复用，解决了队头阻塞问题。

### http/2

[跳转至华为对 http/2 的解释](https://info.support.huawei.com/info-finder/encyclopedia/zh/HTTP--2.html)
[其他的 http/2 分析文章](https://web.dev/performance-http2/)

**对比 http/1.x**

- 没有对 http 协议的应用语义做出改动，依然使用请求方法、状态码、头字段等。
- 传输格式从文本格式转为二进制
- 头部压缩，因为很多头部都是类似的
- 实现了多路复用 multiplexing，可以在同一个 tcp 连接中并行的交换消息
- 服务器推送，可以在请求前发送数据，因为服务器可能知道你需要这些数据

### xss 跨站脚本攻击

[美团技术团队](https://tech.meituan.com/2018/09/27/fe-security.html)

通过将恶意脚本注入到网页中，从而在网页上执行脚本，可以获取敏感信息。

不是只有 script 标签才能触发脚本执行，还可以是 html 元素的属性，例如在 a 标签的 href 属性中使用 JavaScript 协议。

```html
<a href="javascript: alert(1)">link</a>
```

XSS 的本质是：恶意代码未经过滤，与网站正常的代码混在一起；浏览器无法分辨哪些脚本是可信的，导致恶意脚本被执行。

## CSS

### 正常布局流

块级元素宽度与父元素一致，每个块级元素都会另起一行。

内联元素在父元素还有空间的情况下，都会在同一行显示，如果空间不够会换行（如果单个单词在默认设定下无法换行会溢出）。

正常布局流还会出现外边距合并。

### 弹性盒子

弹性盒子是按照行或列布局的一维布局方式，元素即可以膨胀填充额外空间，也可以收缩适应更小的空间。

弹性盒子沿着两个轴进行布局。

<figure>
  <img src="/assets/images/flex_terms.png">
</figure>
我们首先需要了解下面这几个术语。

- flex container：设置了`display: flex | inline-flex`的父元素

- flex item：flex container 的子元素

- 主轴 main axis

- 主轴开始/结束

- main size：flex item 在主轴方向上的大小

- 交叉轴 cross axis

- 交叉轴开始/结束

- cross size：flex item 在交叉轴上的大小

下面介绍在弹性盒子布局中 flex container 可设置的属性

**主轴方向决定列还是行**

`flex-direction: row | row-reverse | column | column-reverse`，row 是默认值

_注意：`flex-direction: row`不一定是从左到右，要根据文字排列方向来决定_

**溢出后使用换行**

默认情况下，flex item 会自适应布局在一行中，这可能造成溢出，可以使用 flex-wrap 属性使其换行。

`flex-wrap: nowrap | wrap | wrap-reverse`，nowrap 是默认值

`no-wrap`表示 flex item 都排列在一行中，可能会超出 flex container。

`wrap-reverse`指交叉轴方向变化。

`flex-flow`是`flex-direction`和`flex-wrap`的缩写。

**主轴方向对齐**

`justify-content`定义了主轴方向上各个 flex item 之间的间隔。

<figure>
  <img src="/assets/images/justify-content.svg">
</figure>
设置 flex item 之间的间隔也可以使用`gap | row-gap | column-gap`，`gap`是`row-gap`和`column-gap`的缩写。

<figure>
  <img src="/assets/images/gap-1.svg">
</figure>

**交叉轴方向对齐**

`align-items`定义了交叉轴方向上的对齐方式，默认值是 stretch。

<figure>
  <img src="/assets/images/align-items.svg">
</figure>
`align-content`定义了多行 flex item 在交叉轴方向上的对齐，只对设置了`flex-wrap: wrap | wrap-reverse`的元素有效。

<figure>
  <img src="/assets/images/align-content.svg">
</figure>

下面介绍 flex item 可设置的属性。

**flex item 的动态尺寸**

`flex-grow`设置一个非负数的无单位的值，表示在主轴方向上的增长系数，默认值为 0。`flex-grow`是对**剩余空间**的再分配，不是整个 flex container 的空间。

<figure>
  <img src="/assets/images/flex-grow.svg">
</figure>

`flex-shrink`设置一个非负的无单位的值，表示收缩系数，一般在 flex item 溢出时使用以防止溢出，值越大收缩的越多。

`flex-basis`设置了 flex item 在主轴方向上的初始大小，默认值是 auto。

`flex`是上面三者的缩写。

**flex item 的顺序**

`order`可以改变 flex item 的顺序，默认值是 0。

<figure>
  <img src="/assets/images/order.svg">
</figure>
### 网格

网格是一种二维的布局方式，通过网格，可以把内容按照列和行的格式进行排版。可以通过设置`display: grid`来定义一个网格。

**网格轨道**

可以简单理解为定义行和列，给网格容器设置`grid-template-rows`定义行，设置`grid-template-columns`定义列。

**网格线**

定义网格轨道时创建了网格线，网格线也可以自己命名，不设置名称就是数字编号。

<figure>
  <img src="/assets/images/learn-grids-inspector.png">
</figure>

**网格间距**

`row-gap`和`column-gap`可以设置行和列的间距，`gap`是两者的简写形式。

**基于网格线放置网格项**

```html
<div class="grid-demo-app">
  <div class="grid-demo-header">Header</div>
  <div class="grid-demo-nav">Nav</div>
  <div class="grid-demo-main">Main</div>
  <div class="grid-demo-footer">Footer</div>
</div>
```

```css
.grid-demo-app {
  display: grid;
  grid-template-columns: auto 1fr;
  grid-template-rows: auto 1fr auto;
  gap: 10px;
}
.grid-demo-header {
  grid-column: 1 / 3;
  grid-row: 1;
  background-color: aliceblue;
}
.grid-demo-nav {
  grid-column: 1;
  grid-row: 2 / 4;
  background-color: antiquewhite;
}
.grid-demo-main {
  background-color: aqua;
}
.grid-demo-footer {
  background-color: aquamarine;
}
```

header 的列从 1 号网格线到 3 号网格线，nav 的行从 2 号网格线到 4 号网格线。

<style>
    .grid-demo-app {
        display: grid;
        grid-template-columns: auto 1fr;
        grid-template-rows: auto 1fr auto;
        gap: 10px;
    }
    .grid-demo-header {
        grid-column: 1 / 3;
        grid-row: 1;
        background-color: aliceblue;
    }
    .grid-demo-nav {
        grid-column: 1;
        grid-row: 2 / 4;
        background-color: antiquewhite;
    }
    .grid-demo-main {
        background-color: aqua;
    }
    .grid-demo-footer {
        background-color: aquamarine;
    }
</style>
<div class="grid-demo-app">
    <div class="grid-demo-header">
        Header
    </div>
    <div class="grid-demo-nav">
        Nav
    </div>
    <div class="grid-demo-main">
        Main
    </div>
    <div class="grid-demo-footer">
        Footer
    </div>
</div>

**基于网格区域放置网格项**

`grid-area`是`grid-row-start grid-row-end grid-column-start grid-column-end`的缩写，也可以自定义命名。

例如我们给上面的示例命名 header，nav，main，footer。

`grid-template-areas`可以按照命名来放置元素，`.`表示留空。

```css
.grid-demo-app {
  display: grid;
  grid-template-areas:
    "header header"
    "nav main"
    "nav footer";
  grid-template-columns: auto 1fr;
  grid-template-rows: auto 1fr auto;
  gap: 10px;
}
.grid-demo-header {
  grid-area: header;
  background-color: aliceblue;
}
.grid-demo-nav {
  grid-area: nav;
  background-color: antiquewhite;
}
.grid-demo-main {
  grid-area: main;
  background-color: aqua;
}
.grid-demo-footer {
  grid-area: footer;
  background-color: aquamarine;
}
```

<style>
    .grid-demo2-app {
        display: grid;
        grid-template-areas:
            "header header"
            "nav main"
            "nav footer";
        grid-template-columns: auto 1fr;
        grid-template-rows: auto 1fr auto;
        gap: 10px;
    }
    .grid-demo2-header {
        grid-area: header;
        background-color: aliceblue;
    }
    .grid-demo2-nav {
        grid-area: nav;
        background-color: antiquewhite;
    }
    .grid-demo2-main {
        grid-area: main;
        background-color: aqua;
    }
    .grid-demo2-footer {
        grid-area: footer;
        background-color: aquamarine;
    }
</style>
<div class="grid-demo2-app">
    <div class="grid-demo2-header">
        Header
    </div>
    <div class="grid-demo2-nav">
        Nav
    </div>
    <div class="grid-demo2-main">
        Main
    </div>
    <div class="grid-demo2-footer">
        Footer
    </div>
</div>

### 浮动

浮动的元素会脱离正常的文档流。

### css 居中

[跳转](/centering-in-css)

## 手写代码

### 防抖&节流

防抖 debounce 和节流 throttle 都是控制函数执行频率的优化方式。

**防抖**

防抖是将连续多次的调用合并为一次调用，可以将合并后的唯一一次调用放在开始，也可以放在结尾，这两者都可以实现。

<figure>
  <img src="/assets/images/debounce.webp">
</figure>

一个极简版本的实现如下，此次执行函数是在连续调用请求的最后。

```javascript
function debounce(func, wait) {
  let timer = null;
  return function (...theArgs) {
    const context = this;

    if (timer !== null) {
      clearTimeout(timer);
    }

    timer = setTimeout(() => {
      func.apply(context, theArgs);
      timer = null; // 防止内存泄漏
    }, wait);
  };
}
```

如果要支持在连续调用请求的一开始执行，那么实现如下。

```javascript
function debounce(func, delay, immediate = false) {
  let timeoutID = null;

  return function (...args) {
    if (timeoutID !== null) {
      clearTimeout(timer);
    } else if (immediate === true) {
      func.apply(this, args);
    }

    timeoutID = setTimeout(() => {
      if (immediate === false) {
        func.apply(this, args);
      }
      timeoutID = null;
    }, delay);
  };
}
```

我们可以看到，在 wait 时间内如果再次调用函数是会被忽略的，核心在于重置计时器。

列举一些使用场景：

- 输入框等到连续输入结束后再进行处理

- 连续点击提交表单，只提交 1 次

**节流**

节流与防抖控制频率的方式不同，节流是指在一段时间内最多只执行一次该函数，不管调用请求是否是连续的。

```javascript
function throttle(func, delay, immediate = false) {
  let timeoutID = null;

  return function (...args) {
    if (timeoutID === null) {
      if (immediate === true) {
        func.apply(this, args);
      }

      timeoutID = setTimeout(() => {
        if (immediate === false) {
          func.apply(this, ...args);
        }
        timeoutID = null;
      }, delay);
    }
  };
}
```

使用场景：resize 和 scroll 无需频繁处理

### promise 简单实现

[跳转](/promise-polyfill)

### 用生成器实现 async&await

```javascript
function resolveAfterXSeconds(timeout) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(`resolved after ${timeout} seconds`);
    }, timeout * 1000);
  });
}

async function asyncTest() {
  const result1 = await resolveAfterXSeconds(2);
  console.log(result1);

  const result2 = await resolveAfterXSeconds(3);
  console.log(result2);

  return "done";
}

asyncTest().then((value) => {
  console.log(value);
});

// 依次输出
// resolve after 2 seconds
// resolve after 3 seconds
// done
```

上面的代码可以转换成生成器函数形式。

```javascript
function* genTest() {
  const result1 = yield resolveAfterXSeconds(2);
  console.log(result1);

  const result2 = yield resolveAfterXSeconds(3);
  console.log(result2);

  return "done";
}

const gen = genTest();

const info1 = gen.next();
const promise1 = info1.value;

promise1.then((value) => {
  const info2 = gen.next(value);
  const promise2 = info2.value;

  promise2.then((value) => {
    const info3 = gen.next(value);
    // 此时info3为{value: "done", done: true}
  });
});
```

我们可以总结出它的过程：调用生成器的`next`方法获得对象`{value, done}`，value 就是以前 await 后面的表达式。

注意：这里我们设置的都是 promise，但是 await 后面可以是任意值，不是 promise 的值会被隐式转换。

在 promise 被解决后，将解决值传入生成器，继续调用 next，直到末尾。

现在我们根据总结出的规律，编写一个函数，让生成器函数自动运行。

```javascript
function asyncToGenerator(genFn) {
  return function () {
    const generator = genFn.apply(this, arguments);

    return new Promise((resolve, reject) => {
      function callGenerator(method, arg) {
        let info = null;

        try {
          info = generator[method](arg);
        } catch (error) {
          reject(error);
          return;
        }

        if (!info.done) {
          Promise.resolve(info.value).then(
            (result) => {
              callGenerator("next", result);
            },
            (error) => {
              callGenerator("throw", error);
            }
          );
        } else {
          resolve(info.value);
        }
      }

      callGenerator("next");
    });
  };
}

asyncToGenerator(genTest)().then((value) => {
  console.log(value);
});
```

关键点：

- 调用 async 函数返回 promise
- await 后面的值并不一定是 promise，我们嵌套一层`Promise.resolve`

### 实现深拷贝

主要利用递归的思想实现，但是对于一些内置的对象类型，例如 Date，正则表达式，函数等等，需要特殊处理，这里不一一实现。

为了避免循环引用导致的死循环问题，我们使用一个 weakmap 保存之前已处理结果。

```js
function deepClone(obj, visited = new WeakMap()) {
  if (visited.has(obj)) {
    return visited.get(obj);
  }

  let result;

  if (Array.isArray(obj)) {
    result = [];
    visited.set(obj, result);

    for (let i = 0; i < obj.length; i++) {
      result[i] = deepClone(obj[i], visited);
    }
  } else if (typeof obj === "object" && obj !== null) {
    result = {};
    visited.set(obj, result);

    for (const prop in obj) {
      if (Object.hasOwn(obj, prop)) {
        result[prop] = deepClone(obj[prop], visited);
      }
    }
  } else if (typeof obj === "function") {
    result = function () {
      obj.apply(this, arguments);
    };
    visited.set(obj, result);
  } else {
    return obj;
  }

  return result;
}
```

### 数组去重

使用 Set

```js
function removeDup(array) {
  return [...new Set(array)];
}
```

使用 filter

```js
function removeDup(array) {
  return array.filter((element, index, array) => {
    return array.indexOf(element) === index;
  });
}
```

使用 reduce

```js
function removeDup(array) {
  return array.reduce((acc, current) => {
    if (!acc.includes(current)) {
      acc.push(current);
    }
    return acc;
  }, []);
}
```

### 数字添加千分位

解释`/\B(?=(\d{3})+(?!\d))/g`

- `\B`：表示非单词边界，匹配一个位置，这个位置不是单词边界（即，前面或者后面都是数字或者非单词字符）。这个非单词边界确保了我们不会在数字的开头添加千位分隔符。

- `(?=(\d{3})+(?!\d))`：这是一个零宽度正预测先行断言，它匹配一个位置，这个位置后面的字符满足一定的条件，但是这个条件不会被包含在匹配结果中。具体的条件是：

  - `(\d{3})+`：匹配三个数字，这个匹配重复出现至少一次（"+" 表示匹配一个或多个，即重复出现若干次，这里的“若干次”是至少一次），这个匹配的结果会被保存到一个分组中。

  - `(?!\d)` 零宽度正向否定查找，表示这个位置后面不能跟着数字，也就是说，这个位置后面只能跟着非数字字符或者字符串结尾。这个条件确保我们不会在数字的结尾添加千位分隔符。

```js
function formatNum(num) {
  if (typeof num !== "number") {
    return num;
  }
  const parts = num.toString().split(".");
  const integerPart = parts[0];
  let result = integerPart.replace(/\B(?=(\d{3})+(?!\d))/g, ",");
  if (parts.length > 1) {
    result = `${result}.${parts[1]}`;
  }
  return result;
}
```

## React 相关

### 函数组件与类组件的不同

函数组件利用闭包的特性绑定了渲染时的 props 和 state，react 明确要求不能更改 props（不可变 immutable），state 也不能直接更改，只能通过更新函数。

然而类组件的实例是可变对象（mutable），`this.props`会随着组件的再次渲染修改。

例如下面这个例子，如果在三秒内以不同的 text 重新渲染了 Print 组件，函数组件仍然会显示之前传入的 text，而类组件会显示最新的 text。

```jsx
function Print({ text }) {
  const handleClick = () => {
    setTimeout(() => {
      alert(text);
    }, 3000);
  };

  return <button onClick={handleClick}>print</button>;
}

class Print extends React.Component {
  handleClick = () => {
    setTimeout(() => {
      alert(this.props.text);
    }, 3000);
  };

  render() {
    return <button onClick={this.handleClick}>print</button>;
  }
}
```

如果真的在函数组件中需要使用可变对象，可使用 ref，但是不要在渲染过程中使用 ref（保证组件纯净）。

### 受控组件

表单的数据由 React 组件来管理，在输入后利用 onChange 事件设置它的值从而重新渲染为新值。

好处：数据来源唯一，重置很方便，或者其他内容的值需要跟随它一起变化时也很方便。

缺点：每种可能导致变化的情况都要编写处理函数。

```jsx
function Input() {
  const [text, setText] = useState("");

  const handleChange = (event) => {
    setText(event.target.value);
  };

  return <input value={text} />;
}
```

### 非受控组件

表单的数据由 DOM 处理。

文件输入的值只能通过 File API 进行操作，它始终是非受控组件，可以通过使用 ref 来控制 DOM。

```jsx
function Demo() {
  const inputRef = useRef(null);

  const handleClick = () => {
    inputRef.current.files[0];
  };

  return (
    <>
      <input type="file" ref={inputRef} />
      <button onClick={handleClick}></button>
    </>
  );
}
```

## 资源

[web 性能权威指南（中文版）](https://awesome-programming-books.github.io/http/Web%E6%80%A7%E8%83%BD%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97.pdf)

[web 性能权威指南（英文版）](https://hpbn.co/)

[图解 HTTP](https://awesome-programming-books.github.io/http/%E5%9B%BE%E8%A7%A3HTTP.pdf)

[博客：测试 react 组件](https://www.robinwieruch.de/react-testing-tutorial/)

[博客：比较 hoc 和 hook](https://www.robinwieruch.de/react-hooks-higher-order-components/)

## 备忘

### react 确定 hook 的 dispatcher

调用 hook 首先确定 dispatcher

```js
function useState(initialState) {
  dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

是在 updateFunctionComponent -> renderWithHooks 中根据 current 与 null 比较确定。

```js
if (current === null) {
  dispatcher = HooksDispatcherOnMount;
} else {
  dispatcher = HooksDispatcherOnUpdate;
}
```

### antd 自定义表单控件

如果自定义表单控件需要与 Form 配合使用，需要满足：

- 提供 value 属性（或者与 valuePropName 属性值的同名属性）
- 提供 onChange 事件处理（或者与 trigger 属性值的同名属性进行值变化的事件处理）

### react hook 依赖列表中的函数处理

当 hook 中使用一个外部函数时，如果这个函数或者它调用的函数使用了 state，props 及其衍生品时，最好不要从依赖列表中删除。

例如下面这个例子，`fetchProduct`使用了`productId`。

```jsx
function ProductPage({ productId }) {
  const [product, setProduct] = useState(null);

  async function fetchProduct() {
    const response = await fetch(`http://myapi/product/${productId}`); // 使用了 productId prop
    const json = await response.json();
    setProduct(json);
  }

  useEffect(() => {
    fetchProduct();
  }, []);
  // ...
}
```

方案 1:将这个函数移动到 hook 内部。

```jsx
function ProductPage({ productId }) {
  const [product, setProduct] = useState(null);

  useEffect(() => {
    let ignore = false; // 处理无序响应

    // 把这个函数移动到 effect 内部后，我们可以清楚地看到它用到的值。
    async function fetchProduct() {
      const response = await fetch(`http://myapi/product/${productId}`);
      const json = await response.json();
      if (!ignore) setProduct(json);
    }

    fetchProduct();

    return () => {
      ignore = true;
    };
  }, [productId]); // ✅ 有效，因为我们的 effect 只用到了 productId
  // ...
}
```

方案 2: 尝试将此方法移到组件外。

方案 3:如果函数是纯函数，考虑在渲染期间调用，然后依赖于它的返回值。

方案 4:使用 useCallback 将此函数记忆化，然后依赖于记忆化后的回调函数。

### react render props

[React 旧文档](https://zh-hans.legacy.reactjs.org/docs/render-props.html)
[博客介绍](https://www.robinwieruch.de/react-render-props/)

解决问题：将一个组件已经封装的状态或者行为共享给其他组件。

```jsx
import { useState } from "react";

export default function RenderPropDemo() {
  return (
    <Amount>
      {(amount) => {
        return (
          <>
            <Euro amount={amount} />
            <Pound amount={amount} />
          </>
        );
      }}
    </Amount>
  );
}

function Euro({ amount }) {
  return <p>Euro: {amount * 0.82}</p>;
}

function Pound({ amount }) {
  return <p>Euro: {amount * 0.75}</p>;
}

function Amount({ children }) {
  const [amount, setAmount] = useState(0);

  const onIncrement = () => {
    setAmount((a) => a + 1);
  };

  const onDecrement = () => {
    setAmount((a) => a - 1);
  };

  return (
    <div>
      <span>US Dollar: {amount} </span>

      <button type="button" onClick={onIncrement}>
        +
      </button>
      <button type="button" onClick={onDecrement}>
        -
      </button>
      {children(amount)}
    </div>
  );
}
```

上面的示例中：Amount 组件共享了它的状态给 Euro 和 Pound 组件。

可以使用任何属性名称，例如 render 或者上面的 children 等等，这个属性需要是一个函数。

### 高阶组件

[React 旧的文档](https://zh-hans.legacy.reactjs.org/docs/higher-order-components.html)
[博客介绍](https://www.robinwieruch.de/react-higher-order-components/)

高阶组件和 render props 解决的是同一类问题。

```jsx
function withHighOrderComponent(WrappedComponent) {
  return (props) => {
    // ...
    return <WrappedComponent {...props} />;
  };
}

// 调用这个函数生成了一个新组件
const WithComponentA = withHighOrderComponent(ComponentA);

function App() {
  return <WithComponentA />;
}
```

高阶组件是一个函数，它接受一个组件为参数（还可以添加其他任何参数），返回一个新的包装好的组件。注意不能更改传入的组件，但是可以进行组合。这样的设计模式就重用了 WrappedComponent 的逻辑。

### ref 转发

[react 文档](https://zh-hans.react.dev/reference/react/forwardRef)

可以将 dom 向上暴露给祖先。

```jsx
import { forwardRef, useRef } from "react";

const MyInput = forwardRef((props, ref) => {
  const { label, ...otherProps } = props;
  return (
    <label>
      {label} <input ref={ref} {...otherProps} />
    </label>
  );
});

export default function FowardRefDemo() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput label="Name" ref={inputRef} />
      <button onClick={handleClick}>Edit</button>
    </>
  );
}
```

### react 数据获取

常见进行数据获取的方式：class 组件的`componentDidMount`生命周期、`useEffect`、事件处理。

[博客介绍类组件获取数据](https://www.robinwieruch.de/react-fetching-data/)
[博客介绍 hook 获取数据](https://www.robinwieruch.de/react-hooks-fetch-data/)

### useEffect

`useEffect(setup)`中的 setup 函数只能返回 cleanup 函数，所以下面这样使用不正确，因为 async 函数总是返回一个 promise。

```js
useEffect(async () => {
  // ...
});
```

## 数据结构和算法

### 深度优先遍历

递归实现

```js
function dfs(node) {
  node.visited = true;
  if (node.children) {
    for (const child of node.children) {
      dfs(child);
    }
  }
}
```

栈实现

```js
function dfs(root) {
  const stack = [root];
  while (stack.length > 0) {
    const node = stack.pop();
    node.visited = true;

    for (let i = node.children.length - 1; i >= 0; i--) {
      stack.push(node.children[i]);
    }
  }
}
```

### 广度优先遍历

利用队列实现

```js
function bfs(root) {
  const queue = [root];
  while (queue.length > 0) {
    const node = queue.shift();
    node.visited = true;

    for (const child of node.children) {
      queue.push(child);
    }
  }
}
```

### 二叉树反转

利用递归实现

```js
function reverse(node) {
  if (node !== null) {
    const right = reverse(node.left);
    const left = reverse(node.right);

    node.left = right;
    node.right = left;
  }
  return node;
}
```

也可以使用队列实现

```js
function reverse(root) {
  const queue = [root];
  while (queue.length > 0) {
    const node = queue.shift();
    [node.left, node.right] = [node.right, node.left];
    if (node.left !== null) {
      queue.push(node.left);
    }
    if (node.right !== null) {
      queue.push(node.right);
    }
  }
}
```
