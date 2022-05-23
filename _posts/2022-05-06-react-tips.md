---
title: "React Tips"
excerpt: ""
toc: true
date: 2022-05-06
last_modified_at: 2022-05-06
categories:
  - React
tags:
  - React
---

## 组件

- 组件函数必须以大写字母开头
- JSX 标签必须闭合
- 必须返回 JSX，如果返回多行，要用括号括起来

  ```tsx
  function Header({children: React.ReactNode}) {
    return (
      <h1>{children}</h1>
    )

    // 下面这样不能返回正确内容
    // return
    //  <h1>{children}</h1>

    // 上面的写法会按下面这样执行
    // return;
    //  <h1>{children}</h1>;

    // 这样写也是可以通过的，但是不建议
    // return <h1>
    //  {children}</h1>;
  }
  ```

- 返回的 JSX 只能有一个根结点，如果需要返回多个，可以使用 React Fragment

  ```tsx
  function Lista() {
    return (
      <div>
        <p>aa</p>
        <p>bb</p>
      </div>
    );
  }

  // fragment简短写法
  function Listb() {
    return (
      <>
        <p>aa</p>
        <p>bb</p>
      </>
    );
  }

  // 如果需要设置key属性，可以显示使用fragment
  interface User {
    id: number;
    name: string;
    age: number;
  }

  function Listc({ users }: { users: User[] }) {
    return (
      <div>
        {users.map((user) => {
          return (
            <React.Fragment key={user.id}>
              <p>{user.name}</p>
              <p>{user.age}</p>
            </React.Fragment>
          );
        })}
      </div>
    );
  }
  ```

- Props 解构

  ```tsx
  // props本质就是一个object
  function Component(props) {
    return <h1 {...props}>xxx</h1>;
  }

  // 可以利用解构把属性暴露出来，也可以设置默认值
  // 只有当age={undefined}时，默认值才生效，age={null}或者age={0}都是不生效的
  function Component({ name, age = 19 }) {
    return (
      <h1>
        name: {name} age: {age}
      </h1>
    );
  }
  ```

- 不要把数字直接放在条件渲染&&中

  ```tsx
  function Component({ age }) {
    // 不要这样做，如果age是0，那么会渲染出0
    // return <div>{age && <p>{age}</p>}</div>;

    // 正确写法
    return <div>{age > 0 && <p>{age}</p>}</div>;
  }
  ```

- 组件必须是纯函数

  - 渲染前不要更改已存在的变量
  - 相同的输入参数，返回完全相同的内容

- **不要直接更改 props, state, context**，修改 props 只能由父组件传递新的 props，修改 state 使用 set state 方法

- 设置事件捕获阶段的处理函数 onEventCapture

- 只在最顶层使用 hook（总是在任何 return 前），不要再循环，条件或者嵌套函数中调用 hook

- 只在 React 函数中调用 hook，包括函数组件或者自定义的 hook

- 渲染可以理解为 React 在调用你的组件函数，返回的 JSX 就像是瞬间的一个快照，只不过是可以进行交互的，比如事件处理。渲染时 props，事件处理函数，本地变量等可能用到 state 的地方都是瞬时计算好的。

- 修改 state 永远都是为**下一次**渲染准备的

- 在一次渲染中的 state 是永远不变的，即使事件处理的某些部分是异步的

- 更新状态函数必须也是纯函数，因为这个函数在渲染期间调用，不能在里面设置 state 以及执行副作用等等

- 更新状态是批量的，可以提高效率，也避免了更新一半的疑惑

- 把 state 当成是只读的，不能直接修改，只能用新的代替

- 为什么不建议直接修改 state

  - debug：如果用`console.log`来查看 state 的变化，不会出错
  - 优化：如果使用一个新的来代替，那 React 可以直接使用`prevObj === obj`来判断是否改变
  - 新特性：React 的新特性 state 就是快照，如果直接修改，那就无法使用这个新特性
  - 需求：可能需要一些 undo/redo，那么这样很容易实现
  - 实现简单：React 不用劫持 object 的属性

- 受控组件和非受控组件
  - 受控组件：重要的内容都由父组件通过 props 来指定，优点：更加灵活，缺点：需要父组件来配置 props
  - 非受控组件：重要内容由本地 state 来控制，优点：使用起来简单，缺点：灵活性不够
  - 实际情况应该是 props 和本地 state 混合起来使用
