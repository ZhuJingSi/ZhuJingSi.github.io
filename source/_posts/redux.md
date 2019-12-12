---
title: redux 源码分析
date: 2018-04-06 17:36:13
tags: 
  - react
  - redux
categories: 
  - 前端
---

### Redux 是啥

Redux 是对 [Flux 应用架构](https://www.zhihu.com/question/33864532) 的一种实现，并且引入了大量函数式编程的实现。

简单来讲，**Redux 帮助我们去维护一个数据池，所有组件的部分共享状态都从这个池子里存取**。那有的同学就想了，这不就是个全局变量的概念嘛？是很像，但组件是不能直接操作 Redux Store 里的数据的，必须使用 **`dipatch`** 分发 **`action`** 来触发对应的 **`reducer`** 进而更新数据池。这样遵循这套规则：1) View 层和 Model 层不会耦合得很紧 2) Store 里可能发生的变化是可预测的（看注册了哪些 `action`） 3) 所有状态变化都可监控、可回溯。

这里有一个误区，很多人看到 Redux 就想到 React，觉得它们应该在一起用，甚至觉得 Redux 只能在 React 中用。但其实他俩并没有互相牵扯的关系，Redux 是一种管理数据状态的方法，可以和很多不同框架搭配使用，支持 React、Angular、Ember、jQuery 甚至纯 JavaScript。（当然，Redux 还是和 [React](http://facebook.github.io/react/) 和 [Deku](https://github.com/dekujs/deku) 这类库搭配起来用最好，因为这类库允许你以 state 的形式来描述界面，Redux 通过 action 的形式来发起 state 变化）。React 也不是一定要用 Redux，如果没有很复杂的状态变更，用上 Redux 反而麻烦。如果要将他俩结合使用，通常会使用一个叫 **react-redux** 的库来牵线搭桥，后面会有文章详细介绍。

<!-- more -->

### action、dispatch、reducer

上面提到了三个单词可能有点懵，下面来分别解释一下。

**`store`**：就是一整个数据池，redux 中有且只能有一个 `store`；

**`state`**：某一刻的 `store` 状态称为 `state`；

**`action`**：View 层因用户操作触发的状态变化通知，是一个简单对象，包含 `type` （必需）和 `payload` 两个字段；

**`dispatch`**：是分发 `action` 的唯一方法；

**`reducer`** ：是一个纯函数，接收当前 `state` 并根据 `action` 信息作出修改，最后返回一个全新的 `state`；

所以 Redux 是单向数据流，结合上面介绍的概念，看图就明白了：

{% asset_img redux01.png %}

### 源码

了解了基本概念就可以看源码了。Redux 的源码并不难懂，量也很少

{% asset_img redux02.png %}

- **`utils/`** ：一些工具函数
- **`applyMiddlewar.js`** ：串联多个 `middleware` 改造成一个新的 `dispatch`
- **`bindActionCreators.js`**： 把 `action` 转为同名 key 的对象，且使用 `dispatch` 把每个 `action` 包围起来调用
- **`combineReducers.js`** ：把多个不同 `reducer` 函数合并成一个最终的 `reducer` 函数
- **`compose.js`** ：函数按参数顺序从右到左依次链式调用，其实也是个工具函数
- **`createStore.js`** ：创建一个 Redux Store

> **这些代码 + 注释太长了，不想放在博客里，可以自行去我 Github 上看**，下面就简单描述一下几个主要方法的概况。
>
> *注：这里不贴每个接口的使用示例了，不知道怎么用的可以去网上找*

#### createStore.js

**接收参数**：`reducer`，`preloadedState`，`enhancer`。

`reducer` 不用讲了；`preloadedState` 是初始 state，一般也不用传； `enhancer` 是一个高阶函数，返回一个新的强化过的 store creator，可以根据需要传一些 middleware 进来，通常配合 `applyMiddleware` 使用：`createStore(reducer, applyMiddleware(middlewares…))`。

**返回接口**：`dispatch`，`subscribe`，`getState`，`replaceReducer`。

- `dispatch` 方法是触发 state 变化的唯一途径，其中会调用 reducer 去创建 store，reducer 接收 “当前 state 树” 和 “action”， 并返回一份新的 state 树，并执行所有的监听函数。传入的 action 参数只能是一个普通对象，如果想要传入复杂对象，需要使用相应 enhancer 包裹 createStore 做一些中间操作（因为前面讲过，enhancer 是包裹 createStore 的高阶函数）。`dispatch` 最后返回与传入 action 一摸一样的对象（同样，如果使用了中间件，它可能会魔改 dispatch，使返回值变成其它东西）。

- subscribe` 顾名思义就是“订阅”，它的作用就是维护一个监听函数列表，接收一个参数就是监听函数。`dispatch` 函数中在执行完 reducer 之后会遍历执行每一个监听函数。可以在监听函数中使用 `getState()` 获取最新 state，也可以在监听函数内部调用 `dispatch`。

  *注意：*

  1. 订阅器（subscriptions） 会维护两个监听器列表：`nextListeners` 和 `currentListeners`，即监听列表变化前后的快照。当你在正在调用监听器的时候订阅 (subscribe) 或者去掉订阅（unsubscribe），对当前的 `dispatch` 不会有任何影响，但是对于下一次的 `dispatch`，无论嵌套与否，都会使用订阅列表里最近的一次快照。
  2. 订阅器不应该关注所有 state 的变化，在订阅器被调用之前，往往由于嵌套的 `dispatch` 导致 state 发生多次的改变。
  3. reducer 内部不允许做订阅操作，因为会打乱数据流，reducer 也会不纯。

  

- `getState` 没啥好说的，就是获取当前状态。

- `replaceReducer` 替换当前 reducer。在需要实现代码分离和动态加载 reducer 的时候可能需要用到， 在使用 redux 热加载机制的时候也可能用到。

#### combineReducers.js

**接收参数**：一个 `reducers` 集合。

**返回**：把多个不同 `reducer` 函数合并成一个最终的 `reducer` 函数。

*注意：*一个合格的 `reducer` 不允许返回 undefined，实在不行要么就不做任何修改地返回原来的 state，要么就失去理想地返回一个 null。

#### bindActionCreators.js

**接收参数**：`actionCreators`，`dispatch`。

`actionCreators` 是一个“制造 action“ 的函数的集合，也可以传单个函数（这些函数的作用就是返回一个 `action`）；`dispatch` 你懂的。

**返回**：可以直接调用触发 `dispatch` 的函数（集合，key 不变），就是不用自己手动 `dispatch` 啦～

*注：*惟一使用 `bindActionCreators` 的场景是当你需要把 action creator 往下传到一个组件上，却不想让这个组件觉察到 Redux 的存在，而且不希望把 Redux store 或 `dispatch` 传给它（这在 react-redux 的 connect 方法中常会用到）。

#### applyMiddlewar.js

**接收参数**：多个 middleware。

**返回**：一个新的 createStore 方法。

先上图，一看就明白 applyMiddlewar 是个啥了：

{% asset_img redux03.png %}

这个 applyMiddleware 比较有意思，同样是一个高阶函数（真的是到处都是高阶函数。。。）这块代码不多，我直接贴代码好了，看注释哈。

```javascript
import compose from './compose'
/**
 * 串联多个 middleware 改造成一个新的 dispatch
 */
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      /**
       * 这里使用匿名函数的意义是使 middleware 中每次在调用 dispatch 的时候都重新获取一遍，
       * 因为 middleware 的串联，下面会重新赋值 dispatch，这样做能保证 middleware 中始终是最新的
       */
      dispatch: (...args) => dispatch(...args)
    }
    /**
     * 将当前的 getState 和 dispatch 分发到每个 middleware 中执行一遍
     */
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    /**
     * 每个 middleware 处理一个相对独立的业务需求，返回一个经过包装的新的 dispatch 作为下一个 middleware 的参数
     * 这里把它们的处理结果串联起来，生成一个最终的新的 dispatch
     */
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

再看一下一个经常拿来举例的 middleware：**redux-thunk**，源码巨短。

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

看看 compose 工具函数会发现，这里面的 middleware 其实是一层套一层的：像 `middlewareC(middlewareB(middlewareA(dispatch)))` 酱紫

{% asset_img redux04.png %}

正常情况下，如图左，当我们 `dispatch` 一个 `action` 时，middleware 通过 `next(action)` 一层一层处理和传递 `action` 直到 redux 原生的 `dispatch`。如果某个 middleware 使用 `store.dispatch(action)` 来分发 action，就发生了右图的情况，相当于从外层重新来一遍，假如这个 middleware 一直简单粗暴地调用 `store.dispatch(action)`，就会形成无限循环了，所以 `store.dispatch(action)` 在 middleware 中一般用来处理异步逻辑，在回调里返回普通 `action` 就不会有问题了。

终于整理完了，好累好累 ಥ_ಥ

### 参考

[深入React技术栈](http://www.ituring.com.cn/book/1898)

[redux middleware 详解](https://segmentfault.com/a/1190000004485808)