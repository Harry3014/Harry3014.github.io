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

## javascript基础概念

### 作用域

大致可以分为全局作用域，函数作用域，块作用域。

作用域通过环境记录Environment Record实现，环境记录中记录了变量、函数等标识符，还有一个指针[[OuterEnv]]指向外部的环境记录，直到全局环境的外部环境记录为null，这样就形成了一个链条。

当使用变量和函数时，先在当前自己的环境记录中寻找，如果没有找到则在外部的环境中寻找，一直到全局环境。

如果未找到变量则会抛出引用错误。（注意这里与变量赋值区分）

下面是一个简单的示例

<figure>
  <img src="/assets/images/lexical-environment-simple-lookup.svg">
</figure>

- name变量存在于say函数的环境记录中
- pharse变量存在全局环境记录中，由于say函数的环境记录引用了外部的全局环境记录，所以在say函数中可以访问pharse变量

下面是一个创建内部函数的示例

<figure>
  <img src="/assets/images/closure-makecounter-nested-call.svg" />
</figure>

创建的内部函数的环境记录的外部环境是makeCounter函数的环境记录，所以能够访问count变量。

每次调用函数都会创建新的环境记录，所以即使多次调用makeCounter，他们内部的值也不会互相影响。

```javascript
function makeCounter() {
  let privateCount = 0;
  
  function changeBy(val) {
    privateCount += value;
  }
  
  return {
    increment: function() {
      changeBy(1);
    },
    decrement: function() {
      changeBy(-1);
    },
    value: function() {
      return privateCount;
    }
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

闭包非常有用，因为它可以访问外部函数的作用域，还可以像上面的makeCounter示例一样模拟私有方法。

虽然闭包有诸多优点，但是也有缺点。按照常理，当调用函数完成后，函数内部的变量应该执行垃圾回收，但是由于闭包的存在，内部函数仍然引用着外部环境，所以无法回收。但是js引擎可能会做出一些优化，如果内部函数中没有使用外部函数的变量，那么还是可能会回收的。

例如下面这个例子，内部函数未使用name变量，那么其实可以优化。

```javascript
function makePrinter() {
  const name = "harry";
  return function() {
    console.log("jack");
  }
}

const printer = makePrinter();
printer();
```

如果确定不再使用闭包函数，可以将其设置为null，使其内存可以被回收，否则可能造成内存泄漏。

### var&函数声明提升&let，const暂时性死区

使用var声明变量和函数声明，都会在创建环境记录时一并创建，这叫变量提升和函数声明提升，只是var变量的值为`undefined`。

而let和const声明变量，也会在创建环境记录时一并创建，但是在未运行初始化之前，这些变量都是不可访问的，这一块区域叫暂时性死区。

来看看下面的例子，函数声明提升使得可以在声明之前调用，var变量声明提升使得可以提前访问，但是值为`undefined`。

const变量在未初始化之前都处于暂时性死区，不能访问，否则抛出引用错误。

```javascript
print();

function print() {
  console.log(firstName); // undefined
  console.log(lastName);  // ReferenceError
}

var firstName = "Jack";
const lastName = "Ma";
```

### 对象&原型

大致可以通过下面几种方式创建对象。

- 对象字面量
- 使用构造函数
- Object.create(proto)，这种不是很常用

所有的对象都至少继承一个对象，继承的对象叫做原型prototype，对象的私有属性\_\_proto\_\_指向原型。

在创建函数时会创建原型对象，prototype属性指向原型，原型的constructor属性指向函数。

原型还有它自己的原型，直到Object函数原型的原型为null，这样就构成了原型链。

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

User.prototype.sayHello = function() {
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
  __proto__: User.prototype
}
```

在上面的例子中，user 实例的原型链是 user -> User.prototype -> Object.prototype -> null。

user继承了User函数的原型，所以可以找到sayHello属性进行调用。

User 函数本身也是一个对象，它的原型链是 User -> Function.prototype -> Object.prototype -> null。

User函数继承了Function函数的原型，可以User函数可以调用bind，apply等函数。

### async & await

async函数返回一个promise，promise最终的状态必然是以下两种情况之一：

- 由async函数的返回值fullfill
- async抛出异常reject，抛出异常的原因可能也有两种：
  - await等待的promise被拒绝
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

console中依次输出：

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

到这里async函数结束，返回了undefined，所以promise的状态变为fullfilled，promise的结果为async函数的返回值即undefined。

如果在async函数中没有捕获处理错误，那么promise的状态会变为rejected，结果为reject的原因。

