---
title: setState 源码分析，流程详解
date: 2018-04-09  18:12:53
tags: 
  - react
categories: 
  - 前端
---

### 引子

**`setState`** 方法由父类 `Component` 提供，是 React 组件修改局部状态的方法，使用起来很简单，向 `setState()` 中传入一个 **对象或函数** 就会已有的 state 进行更新并且重新调用 render 方法。

为什么会有两种传参方式呢？举个栗子：

```javascript
...
  handleClickOnLikeButton () {
    this.setState({ count: 0 }) // => this.state.count 还是 undefined
    this.setState({ count: this.state.count + 1}) // => undefined + 1 = NaN
    this.setState({ count: this.state.count + 2}) // => NaN + 2 = NaN
  }
...
```

看起来像是同步执行的代码，为什么结果和预期的不一样呢？

<!-- more -->

上面我们进行了三次 `setState`，但是实际上组件**只会重新渲染一次**，而不是三次。这是因为当你调用 `setState` 的时候，React 并不会马上修改 state。而是把这个对象放到一个更新队列里面，稍后才会从队列当中把新的状态提取出来合并到 `state` 当中，然后再触发组件更新。所以在 React 中多次进行 `setState` 不会带来性能问题。

组件中 state 互相依赖的情况不多，但如果有依赖，就不能这样一顺到底地写了，这时候得传函数进去：

```javascript
...
  handleClickOnLikeButton () {
    this.setState((prevState) => {
      return { count: 0 }
    })
    this.setState((prevState) => {
      return { count: prevState.count + 1 } // 上一个 setState 的返回是 count 为 0，当前返回 1
    })
    this.setState((prevState) => {
      return { count: prevState.count + 2 } // 上一个 setState 的返回是 count 为 1，当前返回 3
    })
    // 最后的结果是 this.state.count 为 3
  }
...
```

React 会把上一个 `setState` 的结果传入这个函数。

### setState 流程

先上图，看个大概，后面讲源码的时候好有个对应：

{% asset_img setState01.png %}

上源码：

```javascript
// src/isomorphic/modern/class/ReactComponent.js
ReactComponent.prototype.setState = function(partialState, callback) {
  ...
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback);
  }
};
```

```javascript
// src/renderers/shared/reconciler/ReactUpdateQueue.js
enqueueSetState: function(publicInstance, partialState) {
  // 获取 ReactComponent 组件对象
  var internalInstance = getInternalInstanceReadyForUpdate(
    publicInstance,
    'setState'
  );

  if (!internalInstance) {
    return;
  }
  // 将 partialState 放入组件的状态队列
  var queue =
    internalInstance._pendingStateQueue ||
    (internalInstance._pendingStateQueue = []);
  queue.push(partialState);

  enqueueUpdate(internalInstance);
},
```

```javascript
// src/renderers/shared/reconciler/ReactUpdates.js
function enqueueUpdate(component) {
  ...
  // 如果不是正处于创建或更新组件阶段，则处理 update 事务
  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  // 如果正在创建或更新组件，则暂且先不处理 update，只是将组件放在 dirtyComponents 数组中
  dirtyComponents.push(component);
}
```

这里第一个重点来了：**`batchedUpdates`**

```javascript
// src/renderers/shared/reconciler/ReactDefaultBatchingStrategy.js
batchedUpdates: function(callback, a, b, c, d, e) {
  ...
  // 批处理最开始时，将 isBatchingUpdates 设为 true，表明正在更新
  ReactDefaultBatchingStrategy.isBatchingUpdates = true;
  ...
  // 以事务的方式处理 updates
  transaction.perform(callback, null, a, b, c, d, e);
  ...
},
```

> 到这里，细心的同学可能会发现，第一个 component 进来到这里 `transaction.perform(callback, null, a, b, c, d, e);` 里的 callback 就是上面的 `enqueueUpdate`，如果对 `transaction` 有了解的话知道这里的 callback 会在事务流程中执行，那不就死循环了吗？其实不会的，因为第一次执行事务时， `RESET_BATCHED_UPDATES` wrapper 会把 `isBatchingUpdates` 设置为 true，再次回到 `enqueueUpdate` 时就会把这第一个 component 扔进 `dirtyComponents`，目的就是这个，因为如果不在 `dirtyComponents` 列表里就不会更新里面的状态。

好的，接下来第二个重点来了：**`transaction`**

> `transaction` 的源码有些复杂，限于现在的水平就没有太深入，这里也不放源码了。

### Transaction（事务）

**`transaction`** 在 setState 的流程中扮演了非常重要的角色，是它决定了 setState 该如何合并，React Transaction.js 中有一张很形象的图：

{% asset_img setState02.png %}

可以看到，回调函数会被几个`wrapper` 包裹。一个 `wrapper` 包含一对 `initialize` 和 `close` 方法。在执行 `perform` 方法之后，首先会依次执行 `initialize` 方法，然后执行 `perform` 方法中的 callback，最后再执行 `close` 方法。

源码中有两个 `wrapper` ：

```javascript
var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

#### RESET_BATCHED_UPDATES

`RESET_BATCHED_UPDATES` 用来管理 `isBatchingUpdates` 状态

```javascript
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    // 事务批更新处理结束时，将 isBatchingUpdates 设为 false
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};
```

#### FLUSH_BATCHED_UPDATES

```javascript
// src/renderers/shared/reconciler/ReactDefaultBatchingStrategy.js
var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};
```

```javascript
// src/renderers/shared/reconciler/ReactUpdates.js
var flushBatchedUpdates = function() {
  // 循环遍历处理完所有 dirtyComponents
  while (dirtyComponents.length || asapEnqueued) {
    if (dirtyComponents.length) {
      ...
      // close 前执行完 runBatchedUpdates 方法，这是关键
      transaction.perform(runBatchedUpdates, null, transaction);
      ReactUpdatesFlushTransaction.release(transaction);
    }
	...
  }
};
```

`FLUSH_BATCHED_UPDATES` 会在一个 `transaction` 的 `close` 阶段运行 **runBatchedUpdates**，从而执行 update。

```javascript
// src/renderers/shared/reconciler/ReactUpdates.js
function runBatchedUpdates(transaction) {
  var len = transaction.dirtyComponentsLength;
  ...
  // 遍历 dirtyComponents
  for (var i = 0; i < len; i++) {
    // dirtyComponents 中取出一个 component
    var component = dirtyComponents[i];

    // 取出 dirtyComponent 中的未执行的 callback,下面就准备执行它了
    var callbacks = component._pendingCallbacks;
    component._pendingCallbacks = null;

    // 执行 updateComponent
    ReactReconciler.performUpdateIfNecessary(
      component,
      transaction.reconcileTransaction
    );

    // 执行 dirtyComponent 中之前未执行的 callback
    if (callbacks) {
      for (var j = 0; j < callbacks.length; j++) {
        transaction.callbackQueue.enqueue(
          callbacks[j],
          component.getPublicInstance()
        );
      }
    }
  }
}
```

**`runBatchedUpdates`** 循环遍历 `dirtyComponents` 数组，主要干两件事：1）执行 `performUpdateIfNecessary` 来刷新组件的 view； 2)然后执行之前阻塞的 callback。下面来看 **`performUpdateIfNecessary`**。

```javascript
// src/renderers/shared/reconciler/ReactCompositeComponent.js
performUpdateIfNecessary: function(transaction) {
  if (this._pendingElement != null) {
    // receiveComponent 会最终调用到 updateComponent，从而刷新 View
    ReactReconciler.receiveComponent(
      this,
      this._pendingElement || this._currentElement,
      transaction,
      this._context
    );
  }
  /**
   * updateComponent 中会执行 React 组件存在期的生命周期方法，
   * 如 componentWillReceiveProps， shouldComponentUpdate， componentWillUpdate，render, componentDidUpdate。 
   * 从而完成组件更新的整套流程。
   */
  if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
    this.updateComponent(
      transaction,
      this._currentElement,
      this._currentElement,
      this._context,
      this._context
    );
  }
},
```

注意上面的 **`updateComponent`**，它讲负责开始执行 React 组件存在期的生命周期方法。

```javascript
// src/renderers/shared/reconciler/ReactCompositeComponent.js
updateComponent: function(
  transaction,
  prevParentElement,
  nextParentElement,
  prevUnmaskedContext,
  nextUnmaskedContext) {
  ...
  var nextState = assign({}, replace ? queue[0] : inst.state);
  for (var i = replace ? 1 : 0; i < queue.length; i++) {
    var partial = queue[i];
    assign(
      nextState,
      // 如果 setState 第一个参数传的是函数的话，就在这里执行啦
      typeof partial === 'function' ?
        partial.call(inst, nextState, props, context) :
        partial
    );
  }
}
```

`setState` 传函数能获取最新 state 的原因就在这儿了，遍历队列的时候发现是个函数就每次都传最新的 `nextState` 进去执行一遍。

### 总结

所以 `setState` 的流程就是这样：

*为了方便表述，以下用 A 代表这个触发了 setState 的组件实例，B 代表其它组件实例*

1. 将 `partialState` 放入 A 的状态队列（如果有多个 `setState` 写在一起，就会被一个个 push 到 A 的状态队列）。
2. 如果当前 B 正在创建或更新过程中，则将 A 放入 `dirtyComponents` 等候，否则调用 `batchedUpdates` 处理。
3. `batchedUpdates` 发起 `transaction.perform()` 事务。
4. wrapper initialize
5. 这个事务流程中的 *anyMethod* 是 `runBatchedUpdates` ，即更新组件状态并走一遍组件生命周期，在 `componentWillMount` 时将 A 状态队列里累积的状态都依次处理了。
6. wrapper close，循环遍历 `dirtyComponents` 并执行 `transaction.perform(runBatchedUpdates, null, transaction);`，于是 A 就被捡起来开始走事务流程，步骤又回到 4。
7. 直到 `dirtyComponents` 里最后一个组件跑完流程，组件都被统一更新了一遍。
8. Game over.

最后附上源码中一张很棒的组件生命周期阶段图：

{% asset_img setState03.png %}

> `setState` 挺精妙的耶有木有！害我看源码看到凌晨四点，哼！(╯‵□′)╯︵┻━┻

### 参考

[setState 之后发生了什么 —— 浅谈 React 中的 Transaction](http://undefinedblog.com/what-happened-after-set-state/)

[组件的 state 和 setState](http://huziketang.mangojuice.top/books/react/lesson10)

[React 源码剖析系列 － 解密 setState](https://zhuanlan.zhihu.com/p/20328570)

[React源码分析4 — setState机制](https://zhuanlan.zhihu.com/p/25882602)