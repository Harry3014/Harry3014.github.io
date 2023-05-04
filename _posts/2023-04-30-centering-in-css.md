---
title: "CSS中的居中"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-04-30
last_modified_at: 2023-04-30
---

参考自[这片文章](https://css-tricks.com/centering-css-complete-guide/)

## 水平居中

### inline 或者 inline-\*的元素

```css
text-align: center;
```

```html
<div class="center-example" style="background-color: lightgreen;">
  <header style="text-align: center;">This text is centered.</header>

  <nav role="navigation" style="text-align: center;">
    <a>One</a>
    <a>Two</a>
    <a>Three</a>
    <a>Four</a>
  </nav>
</div>
```

<div class="center-example" style="background-color: lightgreen;">
  <header style="text-align: center;">This text is centered.</header>

  <nav role="navigation" style="text-align: center;">
    <a>One</a>
    <a>Two</a>
    <a>Three</a>
    <a>Four</a>
  </nav>
</div>

### 设置了宽度的块级元素

```css
margin-left: auto;
margin-right: auto;
```

```html
<div class="center-example" style="background-color: lightgreen;">
  <p style="width: 400px;margin-left: auto;margin-right:auto;">
    I'm a block level element and am centered.
  </p>
</div>
```

<style>
  .center-example {
    display: flow-root;
  }
  .center-example p {
    margin-top: 1em;
    margin-bottom: 1em;
  }
</style>
<div class="center-example" style="background-color: lightgreen;">
  <p style="width: 400px;margin-left: auto;margin-right:auto;">
    I'm a block level element and am centered.
  </p>
</div>

### 使用弹性盒子

```css
.flex-container {
  display: flex;
  justify-content: center;
}
```

```html
<div
  class="center-example"
  style="background-color: lightgreen;display: flex; justify-content: center;"
>
  <p>I'm a block level element and am centered.</p>
</div>
```

<div
  class="center-example"
  style="background-color: lightgreen;display: flex; justify-content: center;"
>
  <p>I'm a block level element and am centered.</p>
</div>

## 垂直居中

### inline、inline-\*、table-cell

```css
vertical-align: middle;
```

### 设置了高度的块级元素

```css
position: absolute;
top: calc(50% - 二分之一元素高度);
```

或者设置`top: 50%`，然后使用`margin-top: -二分之一元素高度`或者`transform: translateY(-50%)`。

### 使用弹性盒子

```css
display: flex;
flex-direction: column;
justify-content: center;
```

## 水平垂直居中

### 设置了高度宽度

```css
position: absolute;
top: calc(50% - 二分之一元素高度);
left: calc(50% - 二分之一元素宽度);
```

或者设置`top: 50%; left: 50%;`，然后使用`margin-top: -二分之一元素高度; margin-left: -二分之一元素宽度`或者`transform: translate(-50%, -50%)`。

### 使用弹性盒子

```css
display: flex;
justify-content: center;
align-items: center;
```
