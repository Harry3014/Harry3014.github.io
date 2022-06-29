---
title: "原型，原型链"
excerpt: ""
date: 2022-06-07
last_modified_at: 2022-06-07
categories:
  - JavaScript
tags:
  - JavaScript
---

我们来说说函数，原型对象，实例对象这三者之间的关系。

函数的属性`prototype`指向原型对象，原型对象的`constructor`属性指向函数，实例对象的`[[Prototype]]`属性指向原型。

举一个例子

```javascript
function User(id, name) {
  this.id = id;
  this.name = name;
}

let user = new User(1, "michael");
```

User 函数的原型对象如下，原型对象本身就是 Object 函数的实例对象，所以原型的`[[Prototype]]`指向`Object.prototype`。

```
User.prototype = {
  constructor: User,
  [[Prototype]]: Object.prototype
}
```

user 实例的结构如下，`user.__proto__ === User.prototype`。

```
user = {
  id: 1,
  name: "michael",
  [[Prototype]]: User.prototype
}
```

每一个实例都有一个属性[[Prototype]]指向原型对象，原型对象也有它的原型对象，直到原型是 null，这样的一种结构叫原型链。

在上面的例子中，user 实例的原型链是 user -> User.prototype -> Object.prototype -> null

User 函数的原型链是 User -> Function.prototype -> Object.prototype -> null
