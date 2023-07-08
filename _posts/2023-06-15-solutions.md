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
  // 使用用户名，密码，或者saml等方式向认证服务器发起renzheng
  login: async (params) => ({}),
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
