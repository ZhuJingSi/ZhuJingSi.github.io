---
title: react-redux 详解
date: 2018-04-07 18:02:52
tags: 
  - react
  - redux
  - react-redux
categories: 
  - 前端
---

### react-redux 是啥

React 有个问题，因为它是单向数据流，所以如果想完成组件之间的复杂通信就需要状态提升，但如果组件嵌套层级比较多了，这个提升和通过 props 层层传递就很恶心了。于是 React 就出了 context 这个接口，类似于全局变量的作用，所有组件都可以访问。但所有人都可以改 context 多不安全啊，得有一套约束和监管力量才行 — 这就是 Redux 了。由于 React 是以 state 的形式来描述界面，所以与 Redux 是天生的一对，用来帮助它俩完美对接的工具就是 `react-redux` 了。

<!-- more -->

当然如果不通过 `react-redux` 直接在 React 中使用 Redux 也是可以的：在最外层容器组件中初始化 `store` ，然后将 `state` 上的属性作为 `props` 层层传递下去。

```javascript
import {createStore, bindActionCreators} from 'redux';

class ParentComp extends Component{
  // 定义 reducer
  const reducer = (state = {count: 0}, action) => {
    switch (action.type){
      case 'INCREASE': return {count: state.count + 1};
      case 'DECREASE': return {count: state.count - 1};
      default: return state;
    }
  }
  // 初始化 store
  const store = createStore(reducer);
  // action creaters
  const actionCreaters = {
    increase: () => ({type: 'INCREASE'}),
    decrease: () => ({type: 'DECREASE'})
  }
  const dispatchWraper = bindActionCreators(actionCreaters, store.dispatch);

  // 搞一些监听器
  store.subscribe(() =>
    // Do something...
  );
  // 把 state 以及用 bindActionCreators 包裹好的状态改变器丢给子组件
  render(){
    return <ChildComp state={store.getState()} {...dispatchWraper} />
  }
}
```

上面的 Demo 看起来很简单，但实际项目中情况会复杂很多，通过 props 传递不是个好的方式。这时候就需要用到 react-redux 提供的 **`Provider`** 和 **`connect`** 方法了。

### Provider

`Provider` 非常简单，最关键的作用就是在 **`context`** 中放入 Redux 的 store，方便子组件获取。

简单介绍一下 `context`:

**`context` 就是一组属性的集合，并被隐式地传递给后代组件。**（我又要说像全局变量了哈哈哈

`context` 使用了几个 API 是需要记住的：

- **`React.withContext`** ：会执行一个指定的上下文信息的回调函数，任何在这个回调函数里面渲染的组件都有这个 `context` 的访问权限。（就像是给子组件提供了一个属性环境一样的。）
- **`getChildContext`** ：和 `React.withContext` 一样的作用，指定的传递给子组件的属性。不过与 `React.withContext` 写法不同，且要先通过 `childContextTypes` 来指定类型，不然会产生错误。
- **`childContextTypes`** ：声明传递给子组件的属性的数据类型。
- **`contextTypes`** ：任何想访问 `context` 里面的属性的组件都必须显式地指定一个 `contextTypes` 的属性。如果没有指定该属性，那么组件通过 this.context 访问属性将会出错。

**`Provider`** 就是使用了 `getChildContext` 将 store 绑到 context 上使子元素都可以访问到。

> `React.withContext` 这个接口源码里没有找到，可能已经没了吧。。。用 `getChildContext` 就好了。
>
> 我上面只是拎出概念讲解，太抽象不明白的话可以看这篇 [React context](https://segmentfault.com/a/1190000002878442)，里面有清楚的 Demo 示例。
>
> *注意：context 相关内容在将来仍有可能会发生变动，所以不建议直接使用在生产环境中。*

### Connect

**`connect`** 是重头戏，看了很久源码都没懂的！气死了！（到现在也没能通读源码，但好在理清楚了“它是谁、干什么的、怎么用它”）。

首先 `connect` 还是一个**高阶函数**，用来**装饰 React 组件**。

使用的时候容器组件会被**包裹在 `Provider` 组件下面**，这样这些组件就可以获得 `Provider` 挂在 `context` 上的 state 了。 `connect` 作为高阶函数包裹这些容器组件就可以接收到 state 并：**1）将指定 state 和 action 作为 props 绑定到组件上方便调用 2） 帮助组件订阅监听 state 的变化**

```javascript
class MyComp extends Component {
  // ...
}
const Comp = connect(...args)(MyComp);
```

`connect` 方法接收四个参数，下面一一讲解。

```javascript
connect([mapStateToProps], [mapDispatchToProps], [mergeProps],[options])
```

#### mapStateToProps(state, ownProps) : stateProps

这个函数允许我们**将 `store` 中的数据作为 `props` 绑定到组件上**。

接收的第一个参数是我们需要从 Redux 中提取的状态，第二个参数是组件本身的 props。不必将 Redux 中所有的 state 数据都传进组件，可以结合 `ownProps` 进行筛选，传入需要的最少属性。

```javascript
const mapStateToProps = (state, ownProps) => {
  return {
    user: _.find(state.userList, {id: ownProps.userId})
  }
}

class MyComp extends Component {
  static PropTypes = {
    userId: PropTypes.string.isRequired,
    user: PropTypes.object
  };
  
  render(){
    return <div>用户名：{this.props.user.name}</div>
  }
}

const Comp = connect(mapStateToProps)(MyComp);
```

**当 `state` 变化，或者 `ownProps` 变化的时候，`mapStateToProps` 都会被调用**，计算出一个新的 `stateProps`，（在与 `ownProps` merge 后）更新给 `MyComp`。

#### mapDispatchToProps(dispatch, ownProps): dispatchProps

这个函数允许我们**将 `action` 作为 `props` 绑定到组件上**。

```javascript
import {bindActionCreators} from 'redux';

const mapDispatchToProps = (dispatch, ownProps) => {
  return bindActionCreators({
    increase: action.increase,
    decrease: action.decrease
  }, dispatch);
}

class MyComp extends Component {
  render(){
    const {count, increase, decrease} = this.props;
    return (<div>
      <div>计数：{this.props.count}次</div>
      <button onClick={increase}>增加</button>
      <button onClick={decrease}>减少</button>
    </div>)
  }
}

const Comp = connect(mapStateToProps， mapDispatchToProps)(MyComp);
```

同样，**当 `ownProps` 变化的时候，`mapDispatchToProps` 也会被调用**，生成一个新的 `dispatchProps`，（在与 `statePrope` 和 `ownProps` merge 后）更新给 `MyComp`。**注意，`action` 的变化不会引起上述过程，默认 action 在组件的生命周期中是固定的**。

### mergeProps(stateProps, dispatchProps, ownProps): props

**不管是 `stateProps` 还是 `dispatchProps`，都需要和 `ownProps` merge 之后才会被赋给 `MyComp`**。`connect` 的第三个参数就是用来做这件事。通常情况下，你可以不传这个参数，`connect` 就会使用 `Object.assign` 替代该方法。

#### options

最后 `options` 是一些额外的配置项，一般不用去改，本文略过。

#### 监听

上面讲的是我提过的 `connect` 功能的第一块：**将指定 state 和 action 作为 props 绑定到组件上方便调用**。

还有第二块功能，就是：**帮助组件订阅监听 state 的变化**。

被 `connect` 装饰过的组件都会去订阅 state，v5.0 之前存在着一个问题：**父（祖先）子（孙）组件都订阅，子（孙）组件可能受到父（祖先）组件渲染影响，而导致多次渲染**。v5.0 之后 **发布／订阅** 模块重写，解决了之前的问题，主要特点：**增加层级（嵌套）观察者，保证事件通知与组件更新的顺序**。

```javascript
// src/utils/Subscription.js

// 添加层级订阅
addNestedSub(listener) {
  this.trySubscribe()
  return this.listeners.subscribe(listener)
}
// 通知层级订阅
notifyNestedSubs() {
  this.listeners.notify()
}
/**
 * 通过 context 传递 subscription，
 * 组件订阅前检查，如果父（祖先）组件已经订阅，
 * 则将子组件的回调函数 stateChange 订阅到父（向上找到最开始的祖先）组件的观察者，
 * 否则订阅到 redux store
 */
trySubscribe() {
  if (!this.unsubscribe) {
    this.unsubscribe = this.parentSub
      ? this.parentSub.addNestedSub(this.onStateChange) // 订阅到父（向上找到最开始的祖先）组件
      : this.store.subscribe(this.onStateChange) // 订阅到 redux

    this.listeners = createListenerCollection()
  }
}
```

**具体流程**：通过 `context` 传递 `subscription`，组件订阅前检查一下，如果父（祖先）组件已经订阅，则将子组件的回调函数 `stateChange` 订阅到父（向上找到最开始的祖先）组件的观察者，否则订阅到 `redux store`。

```javascript
// src/components/connectAdvanced.js

onStateChange() {
  /**
   * 重新计算当前 Connect 组件的 props，然后和前一次进行比较，如果不是同一个对象的话，就设置 
   * this.selector.shouldComponentUpdate = true, 即更新当前组件
   */
  this.selector.run(this.props)
  
  if (!this.selector.shouldComponentUpdate) {
    this.notifyNestedSubs() // 直接通知子组件更新
  } else {
    // 在 componentDidUpdate 的回调中通知子组件更新
    this.componentDidUpdate = this.notifyNestedSubsOnComponentDidUpdate
    this.setState(dummyState)
  }
}
notifyNestedSubsOnComponentDidUpdate() {
  this.componentDidUpdate = undefined
  this.notifyNestedSubs()
}
```

- 父组件需要更新，则执行 `setState` 重新 `render`， 并在 `ComponentDidUpdate` 时通知子组件更新（触发回调函数）
- 父组件不需要更新，直接通知子组件更新
- 为了保证所有订阅的组件都能获知数据更新，所以需要层层都通知到（通知到你你自己更不更新我管不着，但你得尽职尽责地继续向下通知
- 因父组件重新 render 而被重新 render 过的子组件不会因为得到通知再 render 一遍，因为此时它的数据已经更新，`shouldComponentUpdate` 已经是 false 了

收工 ٩(˃̶͈̀௰˂̶͈́)و

### 参考

[React context](https://segmentfault.com/a/1190000002878442)

[Context，React中隐藏的秘密！](https://github.com/brunoyang/blog/issues/9)

[React 实践心得：react-redux 之 connect 方法详解](http://taobaofed.org/blog/2016/08/18/react-redux-connect/)

[react-redux 工作原理](http://blog.nicksite.me/index.php/archives/421.html?nsukey=INTmTBrXejPyOxl0j7%2FToMB5W0vPZGuc%2B1y2mLX5m1b941DuiyykGIiYmxiiXbvMk%2F8wxv%2F9eccKqitOl%2FHLa46YZ%2BolXY4UQWPuHhWa0%2BVS64%2Flqw2xyRQ2YzOt4I9M%2FbRl01TNfN0uCO5wyIMIuXzYHg%2Fs6S9AeOcdzmfMuGgeXaaHqHH8nI9wmaeIPa9oyWjBF%2FBcCbfLbR1tygVDGg%3D%3D)