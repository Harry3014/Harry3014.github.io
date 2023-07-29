---
title: "css animation和transition"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-07-27
last_modified_at: 2023-07-27
categories:
  - Frontend
tags:
  - Frontend
---

## animation

[MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@keyframes)

[web.dev](https://web.dev/learn/css/animations/)

[使用 animation](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_animations/Using_CSS_animations)

[在线示例](https://stackblitz.com/edit/js-gkedmc?file=style.css)

animation 属性是 animation-name，animation-duration, animation-timing-function，animation-delay，animation-iteration-count，animation-direction，animation-fill-mode 和 animation-play-state 属性的一个简写属性形式。

`animation` 属性用来指定一组或多组动画，每组之间用逗号相隔。

```css
/* @keyframes duration | easing-function | delay |
iteration-count | direction | fill-mode | play-state | name */
animation: 3s ease-in 1s 2 reverse both paused slidein;

/* @keyframes duration | easing-function | delay | name */
animation: 3s linear 1s slidein;

/* two animations */
animation: 3s linear slidein, 3s ease-out 5s slideout;
```

## transition

[MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition)

[web.dev](https://web.dev/learn/css/transitions/)

[使用 transition](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_transitions/Using_CSS_transitions)

[在线示例](https://stackblitz.com/edit/js-vvt1v9?file=style.css)

transition 属性是 transition-property、transition-duration、transition-timing-function 和 transition-delay 的一个简写属性。

`transition`属性可以被指定为一个或多个 CSS 属性的过渡效果，多个属性之间用逗号进行分隔。

```css
/* Apply to 1 property */
/* property name | duration */
transition: margin-right 4s;

/* property name | duration | delay */
transition: margin-right 4s 1s;

/* property name | duration | timing function */
transition: margin-right 4s ease-in-out;

/* property name | duration | timing function | delay */
transition: margin-right 4s ease-in-out 1s;

/* Apply to 2 properties */
transition: margin-right 4s, color 1s;

/* Apply to all changed properties */
transition: all 0.5s ease-out;

/* Global values */
transition: inherit;
transition: initial;
transition: unset;
```

## 两者的比较

[参考博客](https://blog.hubspot.com/website/css-transition-vs-animation)

### 相同

- 都依赖于动画性属性，例如 font-family 就不是动画性属性

### 不同

| transition                                           | animation                  |
| ---------------------------------------------------- | -------------------------- |
| 仅仅只是从一个状态过度到另一个状态，不能定义中间步骤 | 可以通过关键帧定义中间步骤 |
| 只能运行一次                                         | 可以运行多次甚至无限次     |
| 需要触发器，无论是 hover 或者 js                     | 可以自动触发               |
