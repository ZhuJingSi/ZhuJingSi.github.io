---
title: lodash 零散记录
date: 2018-04-12 18:31:51
tags: 
  - loads
categories: 
  - 前端
---

> 这篇不是科普帖，只是我自己的学习记录，比较零散，没什么好看的。

<!-- more -->

### `Hash` && `ListCache`

lodash 里有两种用来替代 ES6 Map 方法的实现：`Hash` 和 `ListCache`。

这两个方法都在 .intenal 目录下。

`Hash` 其实是用对象来做缓存，但是对象有一个局限，它的 `key` 只能是字符串或者 `Symbol` 类型，但是 `Map` 是支持各种类型的值来作为 `key`，因此 `Hash` 缓存无法完全模拟 `Map` 的行为，当遇到 `key` 为数组、对象等类型时，`Hash` 就无能为力了。

因此，在不支持 `Map` 的环境下，`lodash` 实现了 `ListCache` 来模拟，`ListCache` 本质上是使用一个二维数组来储存数据。

这两种方法暴露出的接口与 `Map` 相比，除了没有遍历，其它效果都一样。

### `Hash` 存在的价值

从上面的介绍中可以看出来， `ListCache` 和 `Map` 都可以完全替代 `Hash`，那为什么 lodash 还要保留这个方法呢？

因为数据量较大时，对象的存取比 `Map` 或者数组的性能要好。因此，ladash 在能够用 `Hash` 缓存时，都尽量使用 `Hash` 缓存，而能否使用 `Hash` 缓存的关键是 `key` 的类型。

### lodash 实现和自己实现的不同

1. lodash 抽象出了很多工具函数，在不同接口之间共用
2. lodash 做了很多类型判断，考虑了很多边界情况

### 看了源码的接口

1. difference
2. reduce

### 参考

[lodash源码分析之缓存方式的选择](https://juejin.im/post/5a653c676fb9a01c9332c938)