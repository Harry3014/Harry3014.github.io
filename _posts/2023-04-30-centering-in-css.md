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
.parent {
  position: relative;
}

.child {
  height: 100px;
  position: absolute;
  top: calc(50% - 50px);
}
```

或者设置`top: 50%`，然后使用`margin-top: -二分之一元素高度`或者`transform: translateY(-50%)`。

### 使用弹性盒子

```css
display: flex;
flex-direction: column;
justify-content: center;
```

## 水平垂直居中

### 使用 flex

无需定宽定高

```css
.parent {
  display: flex;
  justify-content: center;
  align-items: center;

  background-color: antiquewhite;
  width: 500px;
  height: 300px;
}

.child {
  width: 50%;
  height: 50%;
  background-color: aquamarine;
}
```

<style>
  .demo2-parent {
    display: flex;
    justify-content: center;
    align-items: center;

    background-color: antiquewhite;
    width: 500px;
    height: 300px;
  }

  .demo2-child {
    width: 50%;
    height: 50%;
    background-color: aquamarine;
  }
</style>
<div class="demo2-parent">
  <div class="demo2-child"></div>
</div>

### 使用绝对定位和 margin auto

无需定宽定高，`marigin: auto`在垂直方向要起作用的关键是：在垂直方向要能够自动填充高度。

```css
.parent {
  position: relative;

  background-color: antiquewhite;
  width: 500px;
  height: 300px;
}

.child {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  margin: auto;

  width: 50%;
  height: 50%;
  background-color: aquamarine;
}
```

<style>
  .demo3-parent {
    position: relative;

    background-color: antiquewhite;
    width: 500px;
    height: 300px;
  }

  .demo3-child {
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    margin: auto;

    width: 50%;
    height: 50%;
    background-color: aquamarine;
  }
</style>
<div class="demo3-parent">
  <div class="demo3-child"></div>
</div>

### 使用绝对定位和 transform 平移

无需定宽定高

```css
.parent {
  position: relative;

  background-color: antiquewhite;
  width: 500px;
  height: 300px;
}

.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);

  width: 50%;
  height: 50%;
  background-color: aquamarine;
}
```

<style>
  .demo4-parent {
    position: relative;

    background-color: antiquewhite;
    width: 500px;
    height: 300px;
  }

  .demo4-child {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);

    width: 50%;
    height: 50%;
    background-color: aquamarine;
  }
</style>
<div class="demo4-parent">
  <div class="demo4-child"></div>
</div>

### 使用 grid 配合 margin auto

只有一列格子

```css
.parent {
  display: grid;

  background-color: antiquewhite;
  width: 500px;
  height: 300px;
}

.child {
  margin: auto;

  width: 50%;
  height: 50%;
  background-color: aquamarine;
}
```

<style>
  .demo7-parent {
    display: grid;

    background-color: antiquewhite;
    width: 500px;
    height: 300px;
  }

  .demo7-child {
    margin: auto;

    width: 50%;
    height: 50%;
    background-color: aquamarine;
  }
</style>
<div class="demo7-parent">
  <div class="demo7-child"></div>
</div>

### 使用绝对定位和 top left margin

需要知道内容的高度和宽度，如果内容的`box-sizing`不是`border-box`，要考虑`padding`和`border`

```css
.parent {
  position: relative;

  background-color: antiquewhite;
  width: 500px;
  height: 300px;
}

.child {
  position: absolute;
  top: 50%;
  left: 50%;
  margin-top: -75px;
  margin-left: -125px;

  width: 250px;
  height: 150px;
  background-color: aquamarine;
  padding: 20px;
  box-sizing: border-box;
}
```

<style>
  .demo5-parent {
    position: relative;

    background-color: antiquewhite;
    width: 500px;
    height: 300px;
  }

  .demo5-child {
    position: absolute;
    top: 50%;
    left: 50%;
    margin-top: -75px;
    margin-left: -125px;

    width: 250px;
    height: 150px;
    background-color: aquamarine;
    padding: 20px;
    box-sizing: border-box;
  }
</style>
<div class="demo5-parent">
  <div class="demo5-child">child with padding</div>
</div>

### 使用绝对定位和 top left 和 calc 函数

需要知道内容的高度和宽度，如果内容的`box-sizing`不是`border-box`，要考虑`padding`和`border`

```css
.parent {
  position: relative;

  background-color: antiquewhite;
  width: 500px;
  height: 300px;
}

.child {
  position: absolute;
  top: calc(50% - 75px);
  left: calc(50% - 125px);

  width: 250px;
  height: 150px;
  background-color: aquamarine;
  padding: 20px;
  box-sizing: border-box;
}
```

<style>
  .demo6-parent {
    position: relative;

    background-color: antiquewhite;
    width: 500px;
    height: 300px;
  }

  .demo6-child {
    position: absolute;
    top: calc(50% - 75px);
    left: calc(50% - 125px);

    width: 250px;
    height: 150px;
    background-color: aquamarine;
    padding: 20px;
    box-sizing: border-box;
  }
</style>
<div class="demo6-parent">
  <div class="demo6-child">child with padding</div>
</div>
