---
title: RxJs 理解
date: 2019-12-05 17:36:13
tags: 
  - rx
  - rxjs
categories: 
  - 前端
---

**RxJS** 是 Reactive Extensions for JavaScript 的缩写，起源于 Reactive Extensions，是一个基于可观测数据流在异步编程应用中的库。**RxJS 是 Reactive Extensions 在 JavaScript 上的实现**，而其他语言也有相应的实现，如 RxJava、RxAndroid、RxSwift 等。

**流**是在时间流逝的过程中产生的一系列事件。它具有时间与事件响应的概念。

<!-- more -->

## 基础概念理解

RxJS 是基于 **观察者模式、迭代器模式和函数式编程** 。因此，首先要对这几个模式有所理解。

### 观察者模式

观察者模式在 Web 中最常见的应该是 DOM 事件的监听和触发。

- **订阅**：通过 addEventListener 订阅 document.body 的 click 事件。
- **发布**：当 body 节点被点击时，body 节点便会向订阅者发布这个消息。

{% asset_img rxjs01.png %}

### 迭代器模式

迭代器模式，提供一种方法顺序访问一个聚合对象中的各种元素，而又不暴露该对象的内部表示。

迭代器模式可以用 JS 提供的 Iterable Protocol 可迭代协议来表示。Iterable Protocol 不是具体的变量类型，而是一种可实现协议。对应到 ES6 中即为 ：[Iterator](http://es6.ruanyifeng.com/#docs/iterator) ，Array、Map、Set 等都原生具备 Iterator 接口，可以通过 Symbol.iterator 属性来获取一个迭代对象，调用迭代对象的 next 方法将获取一个元素对象，如下示例。

```javascript
let arr = ['a', 'b', 'c'];
let iter = arr[Symbol.iterator]();

iter.next() // { value: 'a', done: false }
iter.next() // { value: 'b', done: false }
iter.next() // { value: 'c', done: false }
iter.next() // { value: undefined, done: true }
```

使用迭代器可以实现对三种情况作出不同处理：获取下一个值、无更多值(已完成)、错误处理。

### 函数式编程

函数式编程是声明式编程的体现，首先区分一下命令式编程和声明式编程。

常见的 for 循环即为命令式编程，而使用 map、reduce 等纯函数对数据进行一系列操作则为声明式编程。命令式就是真正实现结果的步骤，声明式的特点是专注于描述结果本身，不关注到底怎么到达结果。

## RxJs

### 观察者模式的表现

RxJS 中含有两个基本概念：**Observables** 与 **Observer**。Observables 作为被观察者，是一个值或事件的流集合；而 Observer 则作为观察者，根据 Observables 进行处理。Observables 与 Observer 之间的订阅发布关系(观察者模式) 如下：

- **订阅**：Observer 通过 Observable 提供的 subscribe() 方法订阅 Observable。
- **发布**：Observable 通过回调 next 方法向 Observer 发布事件。

```javascript
const btn = document.getElementById('btn');

Rx.Observable.fromEvent(btn, 'click')
.subscribe(() => console.log('click'));
```

`fromEvent` 把一个event转成了一个 `Observable`，然后它就可以被 `subscribe `订阅了。

当然，同样可以通过 `unsubscribe` 方法取消订阅：

```javascript
const subscription = stream$.subscribe((value) => {
  console.log('Value', value);
});

setTimeout(() => {
  subscription.unsubscribe();
}, 3000)
```

> 这里注意如果 Observable 中含有 setInterval 之类的轮询操作，需要在 Observable 中 return 一个 clearInterval 的清理函数，再 unsubscribe 的时候 Rx 会自动执行这个函数，从而帮我们取消轮询。

### 迭代器模式的表现

Observable 其实就是数据流 stream 在时间流逝的过程中产生的一系列事件，它具有时间与事件响应的概念。 

把数据管道中每一次数据的流动看成数据中的项，那么数据流就是一个典型的数组，对每一次数据流动的处理就可以映射为数组的迭代器。subscribe 依次接收三种 Observable 的事件，分别为：next（成功）、error（失败）、complete（已完成）：

```javascript
stream$.subscribe(fnValue, fnError, fnComplete)
```

相应的，在 Observable 中通过调用 next()、error()、complete() 来触发这三种事件：

```javascript
const stream$ = Rx.Observable.create((observer) => {
  // observer 是拥有 next、error 和 complete 三个方法的普通对象
  observer.next(1); // 成功
  observer.error('error message'); // 失败
  observer.complete(); // 已完成
});
```

### 函数式编程的表现

Rx 还有一个非常有趣的概念：Operators（操作符）。操作符提供了非常强大的处理 Observable 以及其中流数据的能力，使用上有点像函数式编程中的 map、reduce 等工具方法。RxJS 中有 60+ 个操作符。

举几个常用的例子：

**Rx.Observable.of**
of 可以将普通数据转换成流式数据 Observable，如： Rx.Observable.of(2)。

**Rx.Observable.fromEvent**
除了数值外，RxJS 还提供了关于事件的操作，fromEvent 可以用来监听事件。当事件触发时，将事件 event 转成可流动的 Observable 进行传输。下面示例表示：监听文本框的 keyup 事件，触发 keyup 可以产生一系列的 event Observable。

```javascript
const text = document.querySelector('#text');
Rx.Observable.fromEvent(text, 'keyup')
             .subscribe(e => console.log(e));
```

**Rx.Observable.prototype.map**
map 方法跟我们平常使用的方式是一样的，不同的只是这里是将流进行改变，然后将新的流传出去。

```javascript
Rx.Observable.of(2)
             .map(v => 10 * v)
             .subscribe(v => console.log(v));
```

**Rx.Observable.prototype.mergeMap**
mergeMap 的作用则是将分支流调整回主干上，最终分支上的数据流都经过主干的其他操作，其实也是将流中流进行扁平化。

**Rx.Observable.prototype.switchMap**
switchMap 与 mergeMap 都是将分支流疏通到主干上，而不同的地方在于 switchMap 只会保留最后的流，而取消抛弃之前的流。

**Rx.Observable.prototype.debounceTime**
debounceTime 操作可以操作一个时间戳 TIMES，表示经过 TIMES 毫秒后，没有流入新值，那么才将值转入下一个操作。

> Rx 提供了许多的操作，为了更好的理解各个操作的作用，我们可以通过一个可视化的工具 [marbles 图](http://rxmarbles.com/) 来辅助理解。

### Rx 能解决什么问题

#### 1. 统一数据来源

常见的数据来源有：ajax 请求、WebSocket、localStorage、用户操作。

**数据来源多就导致了获取数据的接口形式不统一，接口形式不统一又导致了逻辑分散难以整合**。比如 WebSocket 的编程方式跟 AJAX 是不一样的，WebSocket是一种订阅，跟主流程很难整合起来，而AJAX相对来说，可以组织得包含在主流程中。有多个数据源影响同一个数据时，对每一个数据源都要分别写一套对应的逻辑，十分啰嗦和不清晰。

#### 2. 统一同步与异步

在前端，经常会碰到同步、异步代码的统一。假设我们要实现一个方法：当有某个值的时候，就返回这个值，否则去服务端获取这个值。当然，可以通过 if/else 来做，但 RxJs 的 BehaviorSubject 显然更加优雅。

`BehaviorSubject` 是一种特殊的 Observable，BehaviorSubject 是 Subject 的变体之一。**BehaviorSubject 的特性就是它会存储“当前”的值。这意味着你始终可以直接拿到 BehaviorSubject 最后一次发出的值。**比起使用一个全局变量或属性来缓存数据，BehaviorSubject 的好处在于它本身也是 Observable，所以异步逻辑对于它来说根本不是问题。

对于这种数据缓存和处理，是 Rx 的拿手好戏，它还有很多功能各异的 Observable/Subject，比如 ReplaySubject 可以用来 "重播"、AsyncSubject 可以在 subject 完成时再给订阅者最终值等等。

#### 3. 统一现在与未来

在业务开发中，我们时常遇到这么一种场景：已过滤排序的列表中加入一条新数据，要重新按照这条规则走一遍。"已过滤排序的列表"是"现在"，"新数据按照这条规则走一遍"是"未来"。

当然你可以将处理的逻辑和数据的获取拼接逻辑解耦：获取数据的逻辑写一套、数据更新的逻辑写一套、数据格式化的逻辑写一套，然后在不同的地方将这三套逻辑自由组合。但这就需要我们在写业务逻辑的时候去考虑数据怎么拼、怎么处理，甚至还要考虑有哪些依赖，比如迁移了这个列表中的一行是不是导航栏括号里的总数也要变一下？另外一个列表是不是要多出一行？。。。太繁琐了。甚至更复杂一些，格式化的过程又依赖了其它 api 的返回，那就更混乱了。

但如果用 Rx 来管理数据流，彻底从数据源处理掉这些依赖和格式化的问题，view 层就只需要单纯地拿到数据展示就行了，以下为伪代码示意：

```javascript
const final$ = start$.merge(patch$) // 合并未来的操作
							.map(filterA).map(sorterA) // 格式化逻辑
```

### Rx 数据流如何应用到视图层

现在主流的MV*框架都基于一个共同的理念：MDV（模型驱动视图），在这个理念下，一切对于视图的变更，首先都应当是模型的变更，然后通过模型和视图的映射关系，自动同步过去。

在这个过程中，我们可能会需要通过一些方式定义这种关系，比如 Angular 和 Vue 中的模板，React 中的 JSX 等等。

在这些体系中，如果要使用 RxJS 的 Observable，都非常简单：

```javascript
data$.subscribe(data => {
  // 这里根据所使用的视图库，用不同的方式响应数据
  // 如果是 React 或者 Vue，手动把这个往 state 或者 data 设置
  // 如果是 Angular 2，可以不用这步，直接把 Observable 用 async pipe 绑定到视图
})

```

### Rx 与 Flux

记得刚开始了解 Rx 的时候，概念还不清晰，问过一个问题："在 Vue 中用了 Rx 是不是就不用用 Vuex 了？"。现在想来其实这个问题是错的，它们不是一个概念，解决的是不同的问题，不能互相替代。

{% asset_img rxjs02.png %}

Redux 和 Vuex 都是对 Flux 架构思维的实现，简单来说就是用来管理项目数据状态的，是顺应数据与视图解偶的趋势、为了中心化控制数据而诞生，通过它可使应用变得可预测、可调式，他们的重点在"管控"。

Rx 是用来管理数据流的，它的着重点在"处理"，虽然它也能实现数据缓存，但无法做到 Flux 那样专业严格的控制整个状态树。

它们可以放在一起用，也可以独立使用，这完全取决于你的需求。如果你的应用数据流很复杂那就用 Rx；如果你的应用规模不小且需要考虑如何更好地在组件外部管理状态，那就用 Flux；如果你的应用足够小也很简单，那么啥都可以不用。

### 与 Promise 的区别

从上面的介绍可以看到，Observable 是一个异步的概念，与 Promise 非常相似，一旦数据达到就可以触发监听，又同样是避免 callback hell、同样是链式调用。然而事实上 Promise 和 Rx 是两种截然不同的模式，不能混为一谈。

{% asset_img rxjs03.png %}

1. Promise 顾名思义，提供的是一个允诺，这个允诺就是在调用 then 之后，它会在未来某个时间段把异步得到的result/error 传给 then 里的函数。

   Rx 不是允诺，它本质上还是由订阅/发布模式引出来的，它的核心思想就是数据响应式，源头是数据产生者，经过一系列的变换/过滤/合并的操作，被数据消费者所使用，数据消费者何时响应，完全取决于数据流何时能流下来。

2. Promise 需要调用 then 或者 catch 才能够执行，catch 是另一种形式的 then，调用 then 或者 catch 之后，它返回一个新的 Promise，这样新的 Promise 也可以同样被调用，所以可以做成无限的 then 链。

   Rx 的数据是否流出不取决于是否 subscribe，也就是说一个 observable 在未被订阅的时候也可以流出数据，在之后它被订阅过后，先前的数据是是否被数据消费者所查知是可设置的，但是数据是否流出还是并不依赖 subscribe。

   Rx 的 observable 被 subscribe 之后，并不是继续返回一个新的 observable，而是返回一个 subscriber，这样用来取消订阅，但是这也导致了链式断裂，所以它不能像 Promise 那样组成无限 then 链。

3. Promise 的数据是一次性流出的，因为 Promise 内部维持着状态，初始化的 pending，转成 resolved 或者 rejected 之后，状态就不可逆转了。

   举例说 promise().then(A).then(B).then(C).catch(D)，数据是顺着链以此传播，但是只有一次，数据从 A 到 B 之后，A 这个 promise 的状态发生了改变，从 pedding 转成了 resolved，那么它就不可能再产生内容了，所以这个 promise 已经不是活动性的了。

   而 Rx 则不同，我们从 Rx 的接口就可以知道，它有 next、error 和 complete，next 可以响应无数次，这也是符合我们对数据响应式的理解，数据在源头被隔三差五的发出，只要源头认为没有流尽（complete）或者出了问题（error），那么数据就可以不断的流到响应者那边。

4. Promise 用 then 链来处理数据，包括对数据进行过滤、合并、变换等操作，它没有真正意义上的数据消费者，then 链的每一个都是数据消费者，所以它非常适合组装流程式，也就是说 A 完成之后做 B，然后 B 完成后去完成 C 这种流程，这些流程可以无穷无尽，没有底的。

   Rx 有数据产生的源头和严格意义的数据消费者，数据可以在中间的操作符里被处理，比如说做过滤，做合并，做节流，变换成新的数据源头等等，可以把它想象成一个完整的数据链，有头也有尾，到了最终消费者那边这个数据流就算到底。

5. Promise 的状态发生改变后，我们如果再想重新从源头开始的话，就需要在后续 then 链递归最开始的那一步，因为后续者是新的 Promise，无法感知源头的那个 Promise。

   而 Rx 的 Observable 可以感知源头，它有类似于 retry、repeat 这种重新开始的运算符，我们可以很方便的链式调用它，而不需要封装成函数再递归。

6. Promise 的 then 链里面，每一行都是同样的角色，也就是 Promise，所以它既可以是源头，也可以是数据处理者。

   Rx 这边的 Observable 还有一些变种，比如说常用的 subject，它可以充当双面角色，可以订阅也可以发消息，这样的话我们还可以用它来做很多封装的工作。

总结来说，**Promise 和 Rx 一个是流程式，一个是数据响应式** 。Promise 可以用来贯串一连串单一的流程，而且这个流程是可以无限的，而 Rx 是用一个数据流来贯串所有操作符，它有一个或多个真正意义上的数据消费者。



## 参考

[RxJS基础教程](https://zhuanlan.zhihu.com/p/34357403)

[构建流式应用—RxJS详解](https://github.com/joeyguo/blog/issues/11)

[流动的数据——使用 RxJS 构造复杂单页应用的数据逻辑](https://github.com/xufei/blog/issues/38)

[DaoCloud 基于 RxJS 的前端数据层实践](https://zhuanlan.zhihu.com/p/28958042)

