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

### localStorage&sessionStorage

它们用于同一个源下的本地存储，并且提供相同的方法。

- setItem(string, string)，如果提供非string，会转成string
- getItem
- removeItem
- clear

localStorage可以长期保存，sessionStorage仅仅在会话期间保存，sessionStorage同一个URL打开不同标签也会创建各自的sessionStorage。

## 常见问题

### 事件循环

为了协调事件，交互，脚本，渲染，网络请求等等，浏览器或者其他用户代理（例如邮件阅读器）使用事件循环模型来管理协调这些内容。

这些事件可以理解为任务，当事件发生时，就将任务放进相应的任务队列，不止一个任务队列，微任务队列是一个特殊的队列，不属于任务队列。

例如script脚本，事件分发，回调，定时器，messageChannel发送消息等等都属于任务。

那么事件循环模型是如何运作的呢？它就是一个持续运行的循环结构。

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

### 浏览器渲染页面

DNS查询得到IP地址

三次握手建立TCP连接

- 浏览器发送SYN，假设序列号是x
- 服务器收到请求，发送SYN，序列号是y，还要发送ACK：x+1
- 浏览器收到ACK，发送ACK：y+1。需要这一次握手的原因：浏览器发送的建立请求因为网络延迟很久才到服务器，服务器无法知晓这个请求是否过期，如果这样就建立了连接，那么是资源的浪费，因为这个请求可能已经过期了。而浏览器是知道这个请求是否已经过期，假设已经过期，那么就不会回复ACK建立连接。

如果使用https协议，还会进行TLS协商以建立安全的连接

发起HTTP请求，通常是获取HTML文件

如果能够成功后去，浏览器会解析HTML

- 构建DOM树（文档对象模型），在解析HTML标签的过程中，没有async或defer的script会阻塞解析HTML，但是一些高优先级的资源会在解析HTML前就发起请求获取，例如css，js文件等，这样能减少阻塞。

  顺带提一下async和defer属性，async会并行请求，defer会延迟到解析完成后请求，在DOMContentLoaded事件触发前。

  load事件是在所有资源都加载完成后才会触发，包括图片等。

- 构建CSSOM，构建CSSOM的速度是非常快的

- 在解析HTML的过程中还会执行script

解析完成后就可以进行渲染

- 将DOM和CSSOM结合成Render树，根据CSS级联规则确定节点的计算样式
- 确定节点的尺寸和位置，第一次称为布局（layout），再次计算称为回流（reflow）
- 绘制（paint）节点到屏幕，例如文本，颜色，边框等等，再次绘制称为重绘（repaint）
- 如果有相互重叠时，还需要合成（compositing），确保以正确的顺序显示

渲染完成后不代表一定能立马进行交互，可能还在执行脚本，此时无法进行交互

## HTTP

首先回顾一些基本的概念。

URL的组成部分

<figure>
  <img src="/assets/images/URI_syntax_diagram.svg.png" />
</figure>

协议://主机:端口/路径?查询#片段

data URL

```
data:[mediatype][;base64],data
```

MIME类型 type/subtype

描述文档文件、字节流的格式和性质，常见的text/plain，application/octet-stream

### 跨源资源共享cors

同源：协议，主机，端口都相同。

**预检请求**

某些类型的跨域请求在发送前需要先发送一个预检请求，服务器处理预检请求告知浏览器是否允许发送真正的跨域请求。

这些需要发送预检请求的跨域请求是：

- 方法限制：不是get，head，post
- 头部限制：包含了一些对于cors来说不安全的头部
- Content-Type限制
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
  - 指定一个来源，如果指定了来源，必须在Vary头部中添加Origin头部用于表明不同Origin会有不同的此头部字段值

- Access-Control-Allow-Methods

  告知浏览器跨域请求允许使用的方法，用于响应预检请求，逗号隔开

- Access-Control-Allow-Headers

  告知浏览器跨域请求允许使用的头部，用于响应预检请求，逗号隔开

- Access-Control-Max-Age

  指定预检请求的响应能够缓存多久，单位秒

除了以上常用的响应头部字段，还有一些其他的用于cors响应的头部字段，但是不常用。

### http缓存

http旨在尽可能多缓存响应，只要满足http规定的缓存条件就能缓存响应，这些条件参考<a href="https://www.rfc-editor.org/rfc/rfc9111.html#name-storing-responses-in-caches" target="_blank">RFC9111</a>。

目前使用比较广泛的缓存控制策略是通过Cache-Control头，但是不使用某些特定头部也是可以被缓存的，只要响应满足上面说的RFC9111规定的条件。

**重用缓存**

在发起请求时重用缓存同样也需要满足一些条件，这些条件参考<a href="https://www.rfc-editor.org/rfc/rfc9111.html#name-constructing-responses-from" target="_blank">RFC9111</a>。

我们比较熟知的可以重用缓存的情况可能是：

- URI要匹配

- 缓存仍然是新鲜的
- 缓存过期，但是经过重新与服务器验证可以使用过期的缓存
- 如果响应含有Vary头，那么Vary指定的头匹配才能使用缓存
- 如果响应含有no-cache指令，必须经过验证才能使用缓存

其实还存在其他可以重用缓存的情况，例如客户端在Cache-Control中使用max-stale指令，表明可以接受过期时间不超过此设定的缓存。

**检查新鲜度**

检查缓存是否新鲜的方式是response_is_fresh = (freshness_lifetime > current_age)。

计算freshness_lifetime的方式按照下面的优先级：

- 如果是共享缓存：Cache-Control的s-maxage指令
- Cache-Control的max-age指令
- Expires头 减掉 Date头
- 以上都没有就属于启发式缓存，使用Date 减掉 Last-Modified，然后取10%

计算current_age稍微有些复杂，可以通俗的理解成缓存的当前寿命。

**重新验证**

虽然缓存中可能存在URI匹配的缓存，但是由于各种原因无法重用缓存，那么可以发起重新验证请求。

如果响应中包含ETag头，在重新验证时使用If-None-Match，如果服务器检查没有修改，可以返回304Not Modified。

如果响应中包含Last-Modified头，在重新验证时使用If-Modified-Since，没有修改返回304。

**Cache-Control常见指令**

可缓存性：public：共享缓存允许被任何对象缓存，private：只允许单个用户缓存，no-cache：仍然可以缓存，只是重用缓存前需要验证，no-store：不缓存

到期：s-maxage：仅仅能指定共享缓存有效期，max-age：缓存有效期，max-stale：客户端愿意使用过期缓存，最大期限

**缓存模式**

最适合缓存的资源是静态资源，如果资源经常变动，建议在资源改动后修改URL，例如添加hash到文件名，或者添加版本号到查询。

### http cookie

可以在响应中使用Set-Cookie头设置cookie，再次向同一服务器发送请求时会附带cookie信息在Cookie头中。

Set-Cookie头包含一些指令，指令之间以分号隔开，例如：

- cookie-name=cookie-value

- 失效时间：expires，max-age
- domain，path
- 设置secure仅仅在https请求发送，设置httponly后无法通过document.cookie访问
- samesite：设置为strict仅仅在同一站点请求发送cookie

可以通过document.cookie读取和写入cookie。

读取document.cookie的值得到cookie1=value1;cookie2=values;...

可以通过设置document.cookie=encodeURIComponent(name) + '=' + encodeURIComponent(value)写入cookie，最好调用encodeURIComponent保持有效格式，如果需要设置指令，以分号隔开。

## CSS基础概念

### 盒子模型

浏览器渲染的每个元素都可以看成是一个盒子结构，完整的盒子模型从内到外由四部分组成：

- content box：用于显示内容
- padding box：内容外部的空白区域，内边距的值不能为负数
- border box：边框
- margin box：盒子和其他元素之间的空白区域

盒子有一个外部显示类型和内部显示类型，可以由display属性指定为块级或内联，内联盒子只使用了盒子模型的部分内容。

默认情况下，width和height属性是作用于content box上的，如果设置了`box-sizing: border-box`（它的默认值是contetn-box），那么padding和border都算作在宽度和高度内。

### 上下外边距折叠

两个盒子的上下外边距相接，那么它们的外边距将会合并成较大的那个，边距合并也是有条件的。

- 设定了float或者position: absolute的不会外边距合并
- 只对块级元素有效，因为margin对内联元素不起作用
- 两个相邻的同级元素
- 父元素与子元素没有内容

这里只是列出大的条件，实际还有很多细节限制条件

### 内联盒子

内联盒子也可以设置高度，宽度，外边距，边框，外边距，但是高度和宽度不起作用，其他三个属性会生效，但是不会影响与其他元素的关系。

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

下面介绍在弹性盒子布局中flex container可设置的属性

**主轴方向决定列还是行**

`flex-direction: row | row-reverse | column | column-reverse`，row是默认值

_注意：`flex-direction: row`不一定是从左到右，要根据文字排列方向来决定_

**溢出后使用换行**

默认情况下，flex item会自适应布局在一行中，这可能造成溢出，可以使用flex-wrap属性使其换行。

`flex-wrap: nowrap | wrap | wrap-reverse`，nowrap是默认值

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

下面介绍flex item可设置的属性。

**flex item 的动态尺寸**

`flex-grow`设置一个非负数的无单位的值，表示在主轴方向上的增长系数，默认值为 0。`flex-grow`是对**剩余空间**的再分配，不是整个 flex container 的空间。

<figure>
  <img src="/assets/images/flex-grow.svg">
</figure>

`flex-shrink`设置一个非负的无单位的值，表示收缩系数，一般在 flex item 溢出时使用以防止溢出，值越大收缩的越多。

`flex-basis`设置了 flex item 在主轴方向上的初始大小，默认值是auto。

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

header的列从1号网格线到3号网格线，nav的行从2号网格线到4号网格线。

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

例如我们给上面的示例命名header，nav，main，footer。

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
