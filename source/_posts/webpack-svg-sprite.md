---
title: webpack 插件 svg-sprite-loader
date: 2017-02-13 13:33:17
tags: 
  - webpack
categories:
  - 前端
---

最近开始看 Vue 了，首先用官方的模版把项目快速搭建起来：

>Vue.js 提供一个[官方命令行工具](https://github.com/vuejs/vue-cli)，可用于快速搭建大型单页应用。该工具提供开箱即用的构建工具配置，带来现代化的前端开发流程。只需几分钟即可创建并启动一个带热重载、保存时静态检查以及可用于生产环境的构建配置的项目。

现在的项目 webpack 几乎是标配了，各种插件好用到爆。现在我们要接触的是一个叫 [**svg-sprite-loader**](https://github.com/kisenka/svg-sprite-loader) 的插件，用来根据导入的 svg 文件自动生成 `symbol` 标签并插入 html，接下来就可以在模版中方便地使用 svg-sprite 技术了。

<!--more-->

### 使用 svg-sprite 的好处

如果不知道 svg-sprite 是什么，可以参考大神张鑫旭的文章：[未来必热：SVG Sprite技术介绍](http://www.zhangxinxu.com/wordpress/2014/07/introduce-svg-sprite-technology/)

- 页面代码清爽
- 可使用 ID 随处重复调用
- 每个 SVG 图标都可以更改大小颜色

### 安装插件

```shell
npm install svg-sprite-loader --save-dev
```

### webpack 配置

装好插件之后，配置 webpack，加入处理 svg 的 loader：

```javascript
{
  test: /\.svg$/,
  loader: 'svg-sprite?' + JSON.stringify({
    name: '[name]',
    prefixize: true,
  }),
}
```

这时候发现还是不行啊，`body` 中并没有看到 `symbol` 标签。

细看发现，还有一个loader 中也处理了 svg 文件：

{% asset_img webpack-svg-sprite01.png %}

那么就把那个 svg 去掉吧，不然下面的 svg-sprite-loader 无法起效... OK，又报错了：

{% asset_img webpack-svg-sprite02.png %}

由于我用了 element.js 作为 UI 框架，它的包里有个 svg 文件需要刚才的 loader 处理，我们把所有的 svg 文件都交由  svg-sprite-loader 处理了，所以这里报了错。

也就是说只有我们自己引入的 svg 文件需要经过 svg-sprite-loader，那么就将这些 svg 统一放到一个目录下，我这里放到了 src/assets/svg/ ，再修改 loader 配置。

上面 loader 匹配正则中的 svg 不用去掉了，给它加上 `exclude` ，不去处理 src/assets/svg/ 路径下的 svg 文件：

```javascript
{
  test: /\.(png|jpe?g|gif)(\?.*)?$/,
  exclude: [
  	path.resolve(__dirname, '../src/assets'),
  ],
  loader: 'url',
  query: {
    limit: 10000,
    name: utils.assetsPath('img/[name].[hash:7].[ext]'),
  }
}
```

下面 svg-sprite-loader 的配置中加上 `include`，只处理 src/assets/svg/ 路径下的 svg 文件：

```javascript
{
  test: /\.svg$/,
  include: [
  	path.resolve(__dirname, '../src/assets'),
  ],
  loader: 'svg-sprite?' + JSON.stringify({
    name: '[name]',
    prefixize: true,
  }),
}
```

重新启动项目，终于在 html 中看到了 `symbol` 标签！

{% asset_img webpack-svg-sprite03.png %}

### 如何使用

配置好了，就可以用了。使用方法很简单，相较于原来插入 svg 图标的方法（img src 或将 svg 整个写入 html），用 svg-sprite 更加简单且清爽：

```javascript
import './assets/svg/target.svg';
```

```html
<svg><use xlink:href="#target" /></svg>
```

嗯，就这样短短一行。`xlink:href` 中传入 svg ID 就好了，由于在上面的配置文件中我们直接使用文件名作为  `symbol` 的 ID，所以这里传入的 ID 即为你想显示的图标的 svg 文件名，记得加上 `#`。

### 自动导入

你会发现，这里要想插入某个图标，都得 ` import`，每用一个都要重复一遍这个流程，太麻烦，那么我们就让 src/assets/svg/ 下的 svg 文件都自动导入吧。

webpack 可以帮我们做到：

```javascript
// requires and returns all modules that match
const requireAll = requireContext => requireContext.keys().map(requireContext);

// import all svg
const req = require.context('./assets/svg', true, /\.svg$/);
requireAll(req);
```

关于 `require.context` 的详细用法，可以参考 webpack 文档：[dynamic requires](https://webpack.github.io/docs/context.html)

这样一来，每当我们修改或增加新的 svg，只要将文件往这个目录下一扔，插件就会自动帮我们生成相应的 `symbol` 标签啦，接下来就只管用吧～♪(´ε｀ )