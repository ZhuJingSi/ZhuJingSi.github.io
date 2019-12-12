---
title: this
date: 2018-01-28 14:29:35
tags: 
  - this
  - 普通函数
  - 构造函数
  - 箭头函数
  - 作用域
categories: 
  - 前端
---

### !试错式编程

js 中的 `this` 是前端开发永远的话题，最近也被问了很多次，干脆这里整理一下好了。

**大道理时间**：我发现做了这么多年的前端，很多人急吼吼地去搞各种框架、构建工具、流程规范、团队管理，如果最终很多基础概念都拎不清，写代码的过程一直是试错的过程，那可能只能算是“一个能干活儿的人”，因为不能从底层出发就不能迸发出更多有趣的想法，世界对你来说也是模糊的。

“试错”是用来帮助你搞清概念的，前端开发方式绝不是“试错式编程”。

<!-- more -->

### 什么是 this

`this` 是 js 中的一个关键字，它代表函数运行时自动生成的一个内部对象，只能在函数内部使用。原则：**在普通函数中，`this` 永远指向正在调用该函数的那个对象**。

以下分四种调用情况讨论：

#### 全局调用

```javascript
this.test; // undefined，this 指向 window 对象
```

#### 普通函数调用

```javascript
function foo() {
  var test = 'foo';
  console.log(this.test);
}
foo(); // 打出 undefined，因为 foo 函数在全局作用域被调用，this 指向 window，但 window 中没有定义 name
```

#### 对象方法调用

```javascript
function foo() {
  var test = 'foo';
  console.log(this.test); // 还是 undefined，就算在其它函数内部被调用，this 还是指向 window
}
const obj = {
  test: 'obj',
  exam: function() {
    console.log(this.test); // 打出 obj，this 指向 obj 对象
    foo();
  }
}
obj.exam();
```

#### 构造函数调用

```javascript
function Dog() {
  this.sound = '汪汪';
  this.bark = function() {
    console.log(this.sound); // 打出'呜呜'，this 指向新对象 dog
  };
}
const dog = new Dog();
dog.sound = '呜呜';
dog.bark();

```

##### 构造函数

关于构造函数又可以引申一些东西出来：

**构造函数与普通函数的区别？**

1. 用 `new` 关键字调用。
2. 函数内部可以使用 `this` 关键字，指向新对象。用 `this` 定义的变量和方法只有用 new 出的实例才能访问到。
3. 不需要 `return` 返回值。默认会返回 `this`（即新的实例对象）。当然，也可以使用 `return`，但根据返回值的类型会有不同的处理结果：
   1. `return` 的是五种简单数据类型：String、Number、Boolean、Null、Undefined。这种情况下，会无视这个 `rertun`，依然返回 `this` 对象。
   2. `return` 的是 Object。这种情况下，不再返回 `this` 对象，而是返回 `return` 语句的返回值。
4. 构造函数首字母大写（建议）

**使用 new 关键字发生了什么？**

1. 创建一个空对象
2. 将构造函数中的 this 指向新创建的对象
3. 将新对象 `_proto_` 属性指向构造函数的 `prototype`，创建对象和原型间的关系
4. 执行构造函数中的代码

### 改变 `this` 的指向

#### `call` 和 `apply`

`call` 和 `apply` 的作用：**当一个对象实例缺少一个函数/方法时，可以用 `call` 和 `apply` 调用其它对象现成的函数/方法。其方式是通过替换其中 `this` 的指向。**

`call` 和 `apply` 作用一样，只是接受参数的方式不一样。`call` 接受**多个单个参数**，`apply` 接受**参数数组**。[参考文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)

```javascript
const cat = {
  who: '猫'
};
function Dog() {
  this.who = '狗';
  this.eat = function() {
    console.log(`${this.who}吃肉`);
  };
};
function Ultraman() {
  this.who = '奥特曼';
  this.beat = function() {
    console.log(`${this.who}打小怪兽`);
  };
};
const dog = new Dog();
const ultraman = new Ultraman();
dog.eat.call(cat); // 猫吃肉
ultraman.beat.call(cat); // 猫打小怪兽
```

这样有天猫想吃鱼或想打小怪兽的时候使用 `call` 或 `apply` 就能具备狗和奥特曼的能力啦。

#### 箭头函数

ES6 出来很久了，箭头函数一定是大家最喜欢用的新特性了，对于箭头函数问得最多当然还是 `this` 的指向了。

在阮一峰老师的《ES6 标准入门》中对箭头函数中的 `this` 是这么描述的：

> 箭头函数有几个使用注意点。
>
> （1）函数体内的`this`对象，就是定义时所在的对象，而不是使用时所在的对象。
>
> （2）不可以当作构造函数，也就是说，不可以使用`new`命令，否则会抛出一个错误。
>
> （3）不可以使用`arguments`对象，该对象在函数体内不存在。如果要用，可以用 rest 参数代替。
>
> （4）不可以使用`yield`命令，因此箭头函数不能用作 Generator 函数。
>
> 上面四点中，第一点尤其值得注意。`this`对象的指向是可变的，但是在箭头函数中，它是固定的。

上面这段话对 `this` 的解释比较模糊，不结合例子是不太能搞清楚的，但其实再往下看，还有一段：

> `this`指向的固定化，并不是因为箭头函数内部有绑定`this`的机制，实际原因是箭头函数根本没有自己的`this`，导致内部的`this`就是外层代码块的`this`。正是因为它没有`this`，所以也就不能用作构造函数。

箭头函数的设计深受纯函数式编程思想的影响，要剔除所有外部状态，所以**在箭头函数中 `this`、`arguments`、`caller` 是统统没有的**。

所以如果能在箭头函数中获得 `this`，那它一定来自父作用域。

```javascript
function A() {
  this.foo = 1;
  this.bar = () => console.log(this.foo);
}
var a = new A();
a.bar(); // 1
```

使用箭头函数，这里有几个长踩的坑：

1. 定义对象的方法

```javascript
const a = {
  foo: 1,
  bar: () => console.log(this.foo)
}
a.bar();  // undefined，对象 a 并不能构成一个作用域，所以再往上到达全局作用域
```

这里如果用普通函数就能打出 1，因为是上文所讲的“对象方法调用”情况。

2. 在原型上使用箭头函数

```javascript
function A() {
  this.foo = 1
}
A.prototype.bar = () => {console.log(this.foo)}
var a = new A();
a.bar(); // undefined，使用 prototype 的话，this 就根据变量查找规则回溯到了全局作用域
```

通过以上说明，我们可以看出，箭头函数除了传入的参数之外，真的是**什么都没有**！如果你在箭头函数引用了 `this`、`arguments` 或者参数之外的变量，那它们一定不是箭头函数本身包含的，而是从父级作用域继承的。



参考文章：

[JavaScript中的普通函数与构造函数](http://www.cnblogs.com/SheilaSun/p/4398881.html)

[少年，不要滥用箭头函数啊](https://jingsam.github.io/2016/12/08/things-you-should-know-about-arrow-functions.html)

[ECMAScript 6 入门](http://es6.ruanyifeng.com/)