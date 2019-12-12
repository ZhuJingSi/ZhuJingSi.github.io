---
title: FP
date: 2018-04-15 18:33:31
tags: 
  - 函数式
categories: 
  - 前端
---

### 函数式编程

**函数式编程** ([Functional programming](https://link.juejin.im/?target=http%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fen.wikipedia.org%2Fwiki%2FFunctional_programming)) 简称 **FP**，并不是什么库或者框架，与过程式编程 (Procedural programming) 相对，而是一种 **编程范式**。FP 通过声明纯函数抽象数据的处理，来避免或尽可能减少函数调用对于外部状态和系统产生的副作用。

函数式编程的实现主要依赖于：

- **声明式编程**
- **纯函数**
- **不可变数据**

<!-- more -->

#### 声明式 && 指令式

- [声明式编程 Declarative programming](https://link.juejin.im/?target=http%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fen.wikipedia.org%2Fwiki%2FDeclarative_programming) 只描述目标的性质，从而抽象出形式逻辑，告诉计算机需要计算什么而不是如何一步步计算。例如正则、SQL、 FP 等。
- [指令式编程 Imperative Programming](https://link.juejin.im/?target=http%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fen.wikipedia.org%2Fwiki%2FImperative_programming) 告诉计算机每一步的计算操作

最简单的，相同的数组处理，使用 for 循环是指令式，用 map 之类的操作是声明式。使用声明式编程，简化了代码，提高了复用率，为重构留有余地。

#### 纯函数

- 函数的输出只与输入有关，相同输入产生的输出一致，并不会依赖外部条件
- 函数调用不会改变函数域以外的状态或者变量，不会对系统产生副作用

#### Immutable

Immutable 即 unchangeable， Immutable data在初始化创建后就不能被修改了，每次对于 Immutable data 的操作都会返回一个新的 Immutable data。

Immutable 实现的原理是 **Persistent Data Structure**（持久化数据结构），也就是使用旧数据创建新数据时，要保证旧数据同时可用且不变。同时为了避免 deepCopy 把所有节点都复制一遍带来的性能损耗，Immutable 使用了 **Structural Sharing**（结构共享），即如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其它节点则进行共享。请看下面动画：

{% asset_img fp01.gif %}

------

#### 柯里化

> 柯里化（currying）是函数式编程的一个概念：只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数。

**lodash-fp**：柯里化的 lodash。

**lodash-fp 特性**：自动 curry 化、迭代优先数据置后、immutable。

1. 将数据的处理逻辑优先传入，不同的逻辑之间可以方便的组合调整
2. 具体数据最后传入，保证输入输出对应

柯里化也有**不好的地方**：难以完善报错机制。如果使用不当（如：参数没有传够），可能导致无法报错（会给你返回一个函数）。

#### compose

compose 也是函数式里的概念，作用是将多个函数组合起来，使它们像管道一样串联，数据从右向左依次流过各函数。compose 符合数学概念中的结合律。compose 有助于实现 pointfree（函数中不直接提及要操作的数据）。

### 练习

> 用纯 FP 方法写一个找出 1000 以内能被 7 整除的所有奇数的平方和的方法

```javascript
/**
* 这边没有抽出中间函数，仅为了验证以下函数式写法的结果的正确性
*/
const test1 = range => {
    let accumulator = 0;
    let index = 0;
    while(++index <= range) {
        accumulator += !(index % 7) && (index % 2) && Math.pow(index, 2);
    }
    return accumulator;
}

/**
* reduce 实现
*/

// 能整除 b 的数字
const aliquot = b => a => {
    if (typeof a !== 'number') return true;
    return !(a % b) && a;
}
// 返回一个数的 b 次方
const square = b => a => {
    if (typeof a !== 'number') return 0;
    return Math.pow(a, b);
}
const sum = b => a => a + b; // 求和
// 创建数组
const createArray = range => Array.from({length: range}).map((v,k) => k + 1);
// 计算当前位置的值
const calculate = (a, ...condition) => {
    return square(condition[2])(!aliquot(condition[1])(aliquot(condition[0])(a)) && a)
}
// 求和
const callback = (pre, cur) => sum(pre)(calculate(cur, 7, 2, 2));
// reduce 逻辑
const reduce = arr => callback => arr.reduce((pre, cur) => callback(pre, cur), 0);

reduce(createArray(1000))(callback);
```

因为写得太丑被嫌弃了，所以经主管提醒改成下面的思路：

```javascript
const compose = (...funcs) => {
    if (funcs.length === 1) return funcs[0];
    return funcs.reduce((pre, cur) => (...args) => pre(cur(...args)));
}
// 创建数组
const range = range => Array.from({length: range}).map((v,k) => k + 1);
// 筛选
const filter = divisible => indivisible => array => {
    return array.filter(number => !(number % divisible) && (number % indivisible));
}
// 次方
const map = power => array => array.map(number => Math.pow(number, power));
// 求和
const sum = array => array.reduce((pre, cur) => (pre + cur), 0);
compose(sum, map(2), filter(7)(2))(range(1000));
```

勤能补拙啦，开熏！

### 参考

[初探 JavaScript 中的函数式编程](https://juejin.im/entry/59196114a22b9d00582429d3)

[Immutable 详解及 React 中实践](https://github.com/camsong/blog/issues/3)