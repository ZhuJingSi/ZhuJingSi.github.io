---
title: setState 再发现
date: 2018-04-11 18:27:25
tags: 
  - react
categories: 
  - 前端
---

之前一篇介绍 `setState` 的文章最后的参考链接中贴上了 [React 源码剖析系列 － 解密 setState](https://zhuanlan.zhihu.com/p/20328570) 这篇文章，开篇有一个引导例子，但对于执行结果细想起来我还是没能搞懂，这篇文章我们就来做做试验。

先回顾这个例子：

```javascript
class Example extends React.Component {
  constructor() {
    super();
    this.state = {
      val: 0
    };
  }
  
  componentDidMount() {
    this.setState({val: this.state.val + 1});
    console.log(this.state.val);    // 第 1 次 log

    this.setState({val: this.state.val + 1});
    console.log(this.state.val);    // 第 2 次 log

    setTimeout(() => {
      this.setState({val: this.state.val + 1});
      console.log(this.state.val);  // 第 3 次 log

      this.setState({val: this.state.val + 1});
      console.log(this.state.val);  // 第 4 次 log
    }, 0);
  }

  render() {
    return null;
  }
};
```

**答案：0、0、2、3**

这里答案不重要，关键是为什么是这样的结果。

<!-- more -->

### 疑问

前一篇文章已经介绍了 `setState` 的流程，`setState` 本身是异步的，所以前两个 log 是 0 没有问题。我的疑问有以下几点：

1. 我们都知道，`setTimeOut` 是异步的，和外面的主线逻辑不会放在同一个 event loop 中执行，所以在进行到 `setTimeOut` 的回调里的时候前两个 `setState` 已经结束了他们的回合，此时的 `this.state.val` 应该是 0+1+1=2 才对，为什么第三个 log 打出来是 2 呢？
2. `setTimeOut` 里的两个 `setState` 是写在一起的，却好像没有合并？两次 log 打印出了不同的结果

### 解答

经过询问主管大大，终于弄懂了一点机制。对以上的疑问相应的解答是：

#### 在同一个 `batchedUpdates` 中对同一个属性赋值会被合并，且会被最后一个的计算方式覆盖。

1. 直接在 `setTimeout` 中打印，可以看到此时的 `this.state.val` 就是 1，并不是预想中的 2

```javascript
this.setState({val: this.state.val + 1}); // 0
this.setState({val: this.state.val + 1}); // 0

setTimeout(() => {
  console.log(this.state.val); // 1
  // this.setState({val: this.state.val + 1});
  // this.setState({val: this.state.val + 1});
}, 0);
```

1. 注意我将第二个 `setState` 里的计算式改成了 `+2`，此时 `setTimeout` 中就打出了 2

```javascript
this.setState({val: this.state.val + 1}); // 0
this.setState({val: this.state.val + 2}); // 0

setTimeout(() => {
  console.log(this.state.val); // 2
  // this.setState({val: this.state.val + 1});
  // this.setState({val: this.state.val + 1});
}, 0);
```

1. 为了验证最后一个会覆盖前面的所有 `setState`，注意我多写了一个 `setState` 并将最后一个 `setState` 里的计算式改回了 `+1`，此时 `setTimeout` 中就打出了 1，验证成功！

```javascript
this.setState({val: this.state.val + 2}); // 0
this.setState({val: this.state.val + 3}); // 0
this.setState({val: this.state.val + 1}); // 0

setTimeout(() => {
  console.log(this.state.val); // 1
  // this.setState({val: this.state.val + 1});
  // this.setState({val: this.state.val + 1});
}, 0);
```

#### 在异步回调中的 `setState` 是同步执行的

`setState` 在主线逻辑上是异步的，但一旦进入类似 `setTimeOut` 这样的异步回调中，就变成同步执行的了。

```javascript
...
componentDidMount() {
  this.setState({val: this.state.val + 2});
  this.setState({val: this.state.val + 2});
  console.log(this.state.val);

  this.setState({val: this.state.val + 1});
  console.log(this.state.val);

  setTimeout(() => {
    this.setState({val: this.state.val + 1});
    console.log(this.state.val);

    this.setState({val: this.state.val + 1});
    console.log(this.state.val);
  }, 0);
}
...
render() {
  console.log('render'+this.state.val); // 为了查看渲染次数以及顺序
}
```

{% asset_img setState2.01.png %}

看结果，可以发现，`setTimeout` 中每 `setState` 一次都会触发一次组件重新渲染。

### 参考

[RFClarification: why is setState asynchronous? #11527](https://github.com/facebook/react/issues/11527#issuecomment-360199710)

这里 react 作者用比较通俗的话解释了一下为什么 setState 要做成异步的，这个等我有时间再翻译一下。