---
title: "关于React需要知道的Javascript知识"
excerpt: ""
toc: true
date: 2022-05-06
last_modified_at: 2022-05-06
categories:
  - Javascript
  - 基础
tags:
  - Javascript
  - React
---

## 模板字面量（模板字符串）
```tsx
const greeting = 'Hello';
const subject = 'World';
console.log(`${greeting} ${subject}!`); // Hello World!

// this is the same as
console.log(greeting + ' ' + subject + '!');

// in React
function Box({className, ...props}) {
  return <div className={`box ${className}`} {...props} />
}
```

## 简短的属性名称
```javascript
const name = "michael";
const age = 18;
const user = {
  name,
  age
};

// this is the same as
use = {
  name: name,
  age: age
};
```

