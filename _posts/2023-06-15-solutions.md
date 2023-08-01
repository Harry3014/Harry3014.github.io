---
title: "架构设计"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-06-15
last_modified_at: 2023-06-15
categories:
  - Frontend
tags:
  - Frontend
---

## 设计理念

component -> hook -> provider -> server

component 调用 hook，hook 使用 provider 提供的 api，provider 可向服务器请求内容

都是在 Refine 这个最外层大组件上设置 provider 是什么，设置到哪里去呢？设置到不同的 Context 上去。

## auth

```ts
const authProvider = {
  // 使用用户名，密码，或者saml等方式向认证服务器发起认证
  login: async (params) => ({}),
  logout: async (params) => ({}),
  // 获取用户资料
  getIdentity: async (params) => ({}),
  // 获取用户权限
  getPermission: async (params) => ({}),
  onError: async (error) => ({}),
  // 检查是否已经认证
  check: async (params) => ({}),
};
```

## live/real-time

### 定义 live context

定义一个 live context，context 的内容提供了三个方法

- subscribe，提供要订阅的内容，还有收到事件后的回调

- unsubscribe

- publish

[refine live context](https://github.com/refinedev/refine/blob/ea30938de7d8ac5d64ea2812472d057f19660ff0/packages/core/src/contexts/live/ILiveContext.ts)

实时事件可以这样定义

```ts
type LiveEvent = {
  channel: string;
  type: "created" | "updated" | "deleted" | "*" | string;
  payload: {
    ids?: number[];
    [x: string]: any;
  };
  date: Date;
};
```

### 提供 context

live context 由 live provider 提供，所以 live provider 可以定位成可替换的内容。

[refine 使用 ably 时提供的 live provider](https://github.com/refinedev/refine/blob/ea30938de7d8ac5d64ea2812472d057f19660ff0/packages/ably/src/index.ts)

[在 Refine 组件上提供 live context](https://github.com/refinedev/refine/blob/ea30938de7d8ac5d64ea2812472d057f19660ff0/packages/core/src/components/containers/refine/index.tsx#L316)

### 使用 context

为了更加方便使用 live context 的内容，可以提供自定义 hook

```ts
export const useSubscription = ({
  params,
  channel,
  types = ["*"],
  enabled = true,
  onLiveEvent,
}: UseSubscriptionProps): void => {
  const liveDataContext = useContext<ILiveContext>(LiveContext);

  useEffect(() => {
    let subscription: any;

    if (enabled) {
      subscription = liveDataContext?.subscribe({
        channel,
        params,
        types,
        callback: onLiveEvent,
      });
    }

    return () => {
      if (subscription) {
        liveDataContext?.unsubscribe(subscription);
      }
    };
  }, [enabled]);
};
```

可以嵌入更多的 hook，例如 useTable，useOne 等等

## data provider

[refine restful data provider](https://github.com/refinedev/refine/blob/ea30938de7d8ac5d64ea2812472d057f19660ff0/packages/simple-rest/src/provider.ts)

## access control

定义 access control context

[refine 源码](https://github.com/refinedev/refine/blob/bf2a0fc1445020e662f1f3ab1e3ac8b8f22e30fb/packages/core/src/contexts/accessControl/IAccessControlContext.ts)

[refine 源码](https://github.com/refinedev/refine/blob/bf2a0fc1445020e662f1f3ab1e3ac8b8f22e30fb/packages/core/src/contexts/accessControl/index.tsx)

组件<CanAccess>，举一个例子

```jsx
<CanAccess resource="employee" action="save" fallback={<p>You cannot access this section</p>} params={{ id: 1 }}>
  <YourComponent />
</CanAccess>
```

hook useCan，返回一个结果表示有无权限，举一个例子

```js
useCan({
  resource: "employee",
  action: "save",
  params: {
    id: 1,
  },
});
```

## TS

### 泛型

在声明时不指定类型，在使用时才指定特定的类型。

### type 和 interface

- type 可以重命名原始类型
- type 不能重新打开添加新属性，可以通过交叉类型，而 interface 可以重复声明

### 反问

- 团队规模
- 对于一个不熟悉 vue 的人有什么建议
- 需求变更后的流程

## 快速加载

- 使用缓存

## 排期

- 需求分析
- 熟悉项目，技术调研
- 写代码
- 自测，配合后台测试
- qa 测试
- 修复
- 部署上线，做好回滚预案
- 预留空间，不熟悉项目 1.2，不熟悉解决方案 1.5，需求随时变化等等因素
- 即时反馈，过程中遇到困难
