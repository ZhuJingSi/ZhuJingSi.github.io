---
title: Cookie、localStorage、sessionStorage
date: 2016-03-09 22:44
tags: 
  - Cookie
  - localStorage
  - sessionStorage
categories: 
  - 前端
---

### localStorage 和 sessionStorage 是什么

**localStorage 和 sessionStorage 都是 HTML5 新的两种浏览器端数据存储方式，统称为 Web Storage。**

### Cookie 是什么

> Cookie（复数形态Cookies），中文名称为“小型文本文件”或“小甜饼”，指某些网站为了辨别用户身份而储存在用户本地终端（Client Side）上的数据（通常经过加密）。
>
> -wikipedia  

<!-- more -->

### Web Storage 为何产生(Cookie 的缺陷)

Cookie 在 1993 年就有了，那么为什么还要弄出个 Web Storage 来呢？因为 Cookie 在某些情况的使用上并不适用或者说有极大的不便：

1. Cookie 会被附加在每个 HTTP 请求中，所以无形中增加了流量。
2. 由于在 HTTP 请求中的 Cookie 是明文传递的，所以安全性成问题。（除非用HTTPS）
3. Cookie 的大小限制在4KB左右。对于复杂的存储需求来说是不够用的。

### 三者区别

#### 生命周期

**Cookie**:  分为非持久 Cookie 和持久 Cookie 两种，非持久 Cookie 由浏览器维护，保存在内存中，浏览器关闭后就消失；持久 Cookie 保存在硬盘中，除非到了设置的过期时间或用户手动清除数据，将会一直保存。默认为非持久 Cookie。

**localStorage**:  没有时间限制，直到 js 中调用接口清除或用户手动清除。

**sessionStorage**:  在用户关闭浏览器窗口或关闭浏览器时就会清空。

#### 作用域

**Cookie**:  同源。

**localStorage**:  同源。

**sessionStorage**:  只在当前窗口共享，即使是同网址的另一个 tab 中都是不能共享的。

#### 存储大小

**Cookie**:  4k左右。

**localStorage**:  5MB 或更大。

**sessionStorage**:  5MB 或更大。

#### 其它

Web Storage 支持事件通知机制，api 接口使用更方便。

sessionStorage 与页面 js 数据对象相比：js 数据对象在页面刷新后就不存在了，但 sessionStorage 还在。

### 浏览器支持情况

要判断浏览器是否支持localStorage可以使用下面的代码：

``` javascript
if(window.localStorage) {
    alert("浏览支持localStorage")
} else {
    alert("浏览暂不支持localStorage")
}
//或者
if(typeof window.localStorage == 'undefined') {
    alert("浏览暂不支持localStorage")
}
```

### localStorage 和sessionStorage 的操作

localStorage 和sessionStorage 都具有相同的操作方法，例如 setItem、getItem 和 removeItem 等。

#### setItem 存储 value

用途：将 value 存储到 key 字段

用法：.setItem( key, value)

``` javascript
sessionStorage.setItem("name", "cisy");
localStorage.setItem("sex", "female");
```

#### getItem 获取 value

用途：获取指定 key 本地存储的值

用法：.getItem(key)

``` javascript
var name = sessionStorage.getItem("name");
var sex = localStorage.getItem("sex");
```

#### removeItem 删除 key

用途：删除指定 key 本地存储的值

用法：.removeItem(key)

``` javascript
sessionStorage.removeItem("name");
localStorage.removeItem("sex");
```

#### clear 清除所有的 key/value

用途：清除所有的 key/value

用法：.clear()

``` javascript
sessionStorage.clear();
localStorage.clear();
```

#### 点操作和[]

``` javascript
var storage = window.localStorage;
storage.key1 = "hello";
storage["key2"] = "world";
console.log(storage.key1); //hello
console.log(storage["key2"]); //world
```

#### storage 事件

storage 还提供了 storage 事件，当键值改变或者clear 的时候，就可以触发 storage 事件。

``` javascript
if(window.addEventListener){
    window.addEventListener("storage",handle_storage,false);
}else if(window.attachEvent){
    window.attachEvent("onstorage",handle_storage);
}
function handle_storage(e){
    if(!e){e=window.event;}
}
```
触发事件的事件对象（e 参数值）有几个属性：

* key : 键值
* oldValue : 被修改前的值
* newValue : 被修改后的值
* url : 页面 url
* storageArea : 被修改的 storage 对象

**注意**：在谷歌浏览器中，需要在不同标签页中修改 storage 才会触发该事件，即 网页A 监听该事件，在 网页B 中修改 localStorage，则 网页A 会触发事件函数。但是在 IE 中，在同个网页修改 localStorage 都会触发该事件。