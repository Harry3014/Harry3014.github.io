---
title: "http缓存"
excerpt: ""
date: 2022-06-20
last_modified_at: 2022-06-20
categories:
  - http
tags:
  - http
---

这篇文章我们来了解一下 http 缓存。

简单说明一下 http 缓存就是把资源存储在一个位置，当客户端再次请求这个资源时，根据不同的缓存策略，有可能会去服务器验证资源是否新鲜，也可能直接读取缓存，而不用从服务器重新下载。

http 缓存的作用：获取资源的时间更快，可以减少网络请求，降低服务器的压力。

是否缓存响应是有很多条件限制的，这里不一一列出了，可以在[RFC](https://www.rfc-editor.org/rfc/rfc9111.html#name-storing-responses-in-caches)中查看。

下面我们介绍常见的几种使用 header 来缓存的情况。

## Expires

在不支持 Cache-Control 的 http1.0 版本，使用 Expires 头部来控制缓存，Expires 只能在响应头中使用。

```http
Expires: Wed, 21 Oct 2015 07:28:00 GMT
```

## Cache-Control

在 http1.1 版本添加，这是一个通用的头部字段，可以在请求头和响应头中使用，也是现在使用最广泛的缓存控制头部。

通过设置指令来控制缓存，可以有多个指令，多个指令使用逗号分开，指令不区分大小写，但是建议使用小写。

请求头和响应头可以使用的指令不是相同的，有些指令只能在请求头或者响应头中使用，具体使用范围可以在[MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)。

下面我们介绍几个常用指令。

### max-age

指令的值是一个时间长度，单位是秒，最长是一年。

```http
Cache-Control: max-age=604800
```

`s-maxage`指令与`max-age`指令类似，但是只能用于 public 类型的缓存，下面我们会介绍 public 和 private 指令。

_如果有 max-age 或者 s-maxage 指令，那么 Expires 头部会被忽略_

### public 和 private

public 指令表示可以缓存到本地和原始服务器中间的代理服务器或者 CDN 等等，可以被多个用户使用，但是应该避免缓存私密信息。

private 指令表示只能缓存到客户端，只能被单个用户使用。

## no-cache

这个指令表示使用缓存之前必须要向原始服务器验证。

## no-store

表示不要缓存这个响应。

## must-revalidate

在缓存是新鲜（fresh）时，可以使用这个缓存，当缓存变得不新鲜（stale）时，那么要向服务器验证。

_应该注意与 no-cache 的区别_

## 没有 Expires 和 Cache-Control

这时候有一种叫启发式缓存的策略。

_可能这种叫法不是很准确，英文是 heuristic caching_

比如使用了`Last-Modified`头部，那么也是有可能缓存这个响应的。

## 使用缓存

当后续发起一个请求，如果命中了一个缓存，那么如何判断这个缓存是否能够使用呢？缓存可能已经过期了。

下面就是判断缓存是否新鲜的公式。

`response_is_fresh = freshness_lifetime > current_age`

那么`freshness_lifetime`和`current_age`又是怎样计算的呢？

### 计算 freshness_lifetime

按照下面的优先级进行计算。

1. `s-maxage`或者`max-age`；
2. `Expires` - `Date`，如果没有`Date`头部，那么就使用收到响应的时间；
3. `Date` - `Last-Modified`的十分之一。

### 计算 current_age

这个算法比较复杂，如果有兴趣可以去[RFC](https://www.rfc-editor.org/rfc/rfc9111.html#name-calculating-age)了解。

## 验证

当发现缓存不新鲜时，也并不是立即就去服务器重新获取这个资源，可以在请求中添加验证字段，如果服务返回的响应表明缓存是新鲜的，那么还是继续使用缓存，而不用重新下载。下面我们就介绍几个验证字段。

### If-Modified-Since

使用`Last-Modified`头部的值，由于可能存在分布式服务器同步文件更新时间的问题，所以建议使用`ETag`和`If-None-Match`。

按照缓存存储的位置可以分为私有缓存和共享缓存，私有缓存只能被单个客户端使用，共享缓存则可以供多个客户端使用，存放在代理服务器，cdn 上的缓存就是共享缓存。

### ETag 和 If-None-Match

`ETag`是一个响应头部，`W/`开头表示使用弱验证，弱验证只要是内容相同就被认为相同，即使可能有一些细微的差别，比如页脚或者广告。

```http
ETag: W/"<etag_value>"
ETag: "<etag_value>"
```

在发起条件验证请求时，`If-None-Match`使用`ETag`的值添加一个请求头，服务器如果发现资源没有更改，那就返回 304。

`ETag`还有一个作用：可以避免空中碰撞，比如我要修改一个资源，但是可能这个资源在我之前已经被修改过了，那这次的修改可能会覆盖上次的修改。这时候可以在请求头部中添加`If-Match`，只有当匹配时才能修改，否则返回 412（先决条件失败）。

还有一些关于 http 缓存值得了解的内容：

1. 现在使用缓存最普遍的请求类型是 GET 请求，但是其他类型也是可以被缓存的；
2. 最普遍缓存的响应是 200，但是也可能缓存重定向或者 404 或者 206 这样的响应；
3. `Vary`响应头部，如果可缓存的响应使用了这个头部，那么如果需要使用这个缓存，那么请求头需要包含匹配`Vary`指定的头部，并且值要相同。比如`Vary: User-Agent`，那么请求的`User-Agent`需要与之前缓存的请求中的一致，这也是处理移动端和桌面客户端使用不同缓存的方式。
