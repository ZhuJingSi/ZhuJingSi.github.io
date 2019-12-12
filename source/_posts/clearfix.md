---
title: clearfix 清除浮动
date: 2016-04-22 19:37:45
tags: 
  - clearfix
  - 清除浮动
  - 闭合浮动
categories:
  - 前端
---

前端样式看起来简单，但真正做前端的同僚都知道，水不是一般的深！今天先从最浅的部分记起－清除浮动的几种方法。

时隔这么久，终于提起兴致再写一篇博客，因为有些东西不写一遍真的记不住。。。_:(´ཀ`」 ∠):_ 

<!--more-->

#### 不在掌控中的 float

在写页面的时候 float 属性出现的概率是很高的，我们总要指挥一个元素：“你！呆到右边去！”这时候一个 float: right; 就解决了，So easy~但却有它不听话的时候，举个例子：

```html
<div class="wrap">
  <div class="float1"></div>
  <div class="float2"></div>
</div>
```

```css
.wrap {
  width: 300px;
  padding: 20px;
  color: white;
  background-color: black;
}
.float1 {
  float: left;
  width: 40%;
  height: 100px;
  background-color: red;
}
.float2 {
  float: right;
  width: 40%;
  height: 100px;
  background-color: blue;
}
```

代码很简单，预想中应该是一红一蓝两个块被包在一个黑块里，然而事实上是这样的： {% asset_img clearfix.png %}

跟预想的不一样啊啊啊 (╯°Д°）╯︵ /(.□ . \)

这是为啥呢？因为 .float1 和 .float2 它俩已经跟爸爸元素 .wrap 不在一个层了，它俩。。。上天了。。。

废话不多说，直接讨论解决办法，让它们重回爸爸的怀抱 (๑>◡<๑) 

#### planA

```html
<div class="wrap">
  <div class="float1"></div>
  <div class="float2"></div>
  <div class="clear"></div>
</div>
```

```css
.clear {
  clear: both;
  line-height: 0;
}
```

在 .wrap 的最后创建一个空子元素 .clear ，然后赋予以上样式。

这是最早出现的清除浮动的方法，但用下来会发现，页面上到处都是 .clear 这个空标签。显然看起来非常不！优！雅！

#### planB

```css
.wrap:after {
  visibility: hidden; 
  display: block; 
  font-size: 0; 
  content: " "; 
  clear: both; 
  height: 0;
}
```

这次清爽多了，html 不用改，利用伪元素 :after 代替之前的空标签，但还是太长了。

#### planC

```html
<div class="wrap">
  <div class="float1"></div>
  <div class="float2"></div>
  <br clear="all">
</div>
```

这个方法比较小众，br 有 clear=“all | left | right | none” 属性

比空标签方式语义稍强，代码量较少，但还是多了个标签，不能忍。。。

#### planD

现在我们把主意打到爸爸元素 .wrap 身上，经过试验，以下属性都会有清除浮动的效果：

* overflow 除了visible 以外的值（hidden，auto，scroll ）
* display (table-cell，table-caption，inline-block)
* position（absolute，fixed） 
* float 除了none以外的值 
* div 改为 fieldset

原理：这些都能触发 [ Block formatting contexts](http://www.w3.org/TR/CSS21/visuren.html#block-formatting)（块级格式化上下文），以下简称 BFC。CSS3里面对这个规范做了改动，称之为：[flow root](http://www.w3.org/TR/css3-box/#block-level0)。

> display:table 本身并不会创建BFC，但是它会产生 [匿名框](http://www.w3.org/TR/CSS21/tables.html#anonymous-boxes) (anonymous boxes)，而匿名框中的 display:table-cell 可以创建新的 BFC。换句话说，触发块级格式化上下文的是匿名框，而不是 display:table。所以通过 display:table 和 display:table-cell 创建的 BFC 效果是不一样的。

缺点：这将导致其布局和相对于浮动的行为等发生一系列的变化，闭合浮动只不过是一系列变化中的一个作用而已。所以为了闭合浮动去改变全局特性，这是不明智的，带来的风险就是一系列的bug。

#### planE

```css
.wrap:before,
.wrap:after {
  content: "";
  display: table;
}
.wrap:after { clear: both; }
```

:before 是用来处理 margin 边距重叠的，由于内部元素 float 创建了BFC，导致内部元素的 margin-top 和 上一个盒子的 margin-bottom 发生叠加。如果这不是你所希望的，那么就可以加上 :before，如果只是单纯的闭合浮动，:after 就够了。