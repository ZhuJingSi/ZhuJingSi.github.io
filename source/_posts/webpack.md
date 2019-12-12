---
title: 浅谈 Webpack 实践
date: 2016-05-21 21:47:37
tags:
  - 构建工具
  - webpack
categories:
  - 前端
---

最近趁着兴致优化了一下手上项目的前端构建流程，正好熟悉了一下 webpack，以下为我对 webpack 的一些浅显认识。

#### 什么是 Webpack

**[Webpack](http://webpack.github.io/) 是一个前端资源加载/打包工具。**

与 [Grunt](http://gruntjs.com/) 和 [Gulp](http://gulpjs.com/) 这两个**自动化工具**不同，Webpack 是个**打包工具**。

具体来说：

Webpack 主要以 entry 文件为入口，对依赖链中的文件进行各种处理（包括 js、图片、css 等多种类型的文件），最终打成一个 js 包。它专注于打包。

Gulp／Gulp 是作为一个 task runner 存在的，它们专注于整个构建流程的串联，必要时 Webpack 的工作可以作为其中一个 task 存在。

<!--more-->

#### 基本流程

主要完成这几个工作：

编译、合并、压缩、打包、Web 服务 => Webpack

文件自动注入(inject)、监听触发任务、修改文件路径 => gulp



Webpack 的配置文件结构主要参考 [Yeoman](http://yeoman.io/) 的模版 [react-webpack](https://github.com/react-webpack-generators/generator-react-webpack#readme)。



配置大体思路：重任都在 Webpack，Gulp 弥补 Webpack 不能实现的功能、串联起整个构建流程。



#### 文件树

```
root  
├── gulp
├── webpack
├── src
│   ├── app
│	│	 ├── components
│	│	 ├── containers
│   ├── core
│   │    ├──bootstrap.js
│   ├── index.html
│   ├── common.scss
│   └── index.module.js
├── webpack.config.js
├── gulpfile.js
├── node_modules
└── package.json

```

以上为整个项目的简略文件树，其中，gulp 目录下为 gulp 的各个 task 文件，webpack 目录下为 不同环境下的 webpack 配置文件，src 下为项目主体。其它的都不用说了。

####  webpack.config.js

先来看一下 webpack.config.js 的内容：

```javascript
const path = require('path');
const args = require('minimist')(process.argv.slice(2));

// List of allowed environments
const allowedEnvs = ['dev', 'dist', 'test'];

// Set the correct environment
let env;
if (args._.length > 0 && args._.indexOf('start') !== -1) {
  env = 'test';
} else if (args.env) {
  env = args.env;
} else {
  env = 'dev';
}
process.env.APP_ENV = env;

/**
 * Build the webpack configuration
 * @param  {String} wantedEnv The wanted environment
 * @return {Object} Webpack config
 */
function buildConfig(wantedEnv) {
  const isValid = wantedEnv && wantedEnv.length > 0 && allowedEnvs.indexOf(wantedEnv) !== -1;
  const validEnv = isValid ? wantedEnv : 'dev';
  const config = require(path.join(__dirname, `webpack/${validEnv}`));
  return config;
}

module.exports = buildConfig(env);
```

上文中 webpack 目录下分别有 default.js、base.js、dev.js、dist.js。default.js、base.js 是基础配置，dev.js、dist.js 分别对应开发环境和生产环境的配置。

这里的 webpack.config.js 会根据环境变量的不同来选择载入相应环境的配置文件。

#### gulpfile.js

```javascript
/**
 *  Welcome to your gulpfile!
 *  The gulp tasks are splitted in several files in the gulp directory
 *  because putting all here was really too long
 */

const gulp = require('gulp');
const wrench = require('wrench');

/**
 *  This will load all js or coffee files in the gulp directory
 *  in order to load all gulp tasks
 */
wrench.readdirSyncRecursive('./gulp').filter((file) => {
  return (/\.(js|coffee)$/i).test(file);
}).map((file) => {
  require(`./gulp/${file}`);
});
```

gulpfile.js 则会载入 gulp 目录下的所有 task 配置。

#### webpack 各配置文件

##### default.js

定义了一些全局变量、loader 的配置、开发生产环境共用的 webpack 插件。是 base.js 的基础。

```javascript
const path = require('path');
const srcPath = path.join(__dirname, '/../src');
const dfltPort = 8012;
const webpack = require('webpack');
const HappyPack = require('happypack');
const happyThreadPool = HappyPack.ThreadPool({ size: 5 });
const BowerWebpackPlugin = require('bower-webpack-plugin');

const DEFAULT_ENV = {
  API_URL: '"http://192.168.1.30"',
};

const CURRENT_ENV = Object.assign({}, DEFAULT_ENV);

if (process.env.API_URL) {
  CURRENT_ENV.API_URL = JSON.stringify(process.env.API_URL);

  if (process.env.API_URL === '/') {
    CURRENT_ENV.API_URL = JSON.stringify('.');
  }
}

function getDefaultModules() {
  return {
    loaders: [{
      test: /jquery.js$/,
      loader: 'expose?jQuery',
    }, {
      test: /\.js$/,
      include: [srcPath],
      exclude: /(node_modules|bower_components|helpers\/lib|\.spec\.js$)/,
      loaders: ['happypack/loader?id=js'],
    }, {
      test: /\.html$/,
      loaders: ['happypack/loader?id=html'],
    }, {
      test: /\.css$/,
      loader: 'style-loader!css-loader!postcss-loader',
    }, {
      test: /\.scss/,
      loaders: ['happypack/loader?id=scss'],
    }, {
      test: /\.svg$/,
      loader: 'svg-sprite?' + JSON.stringify({
        name: '[name]',
        prefixize: true,
      }),
    }],
  };
}
module.exports = {
  srcPath,
  publicPath: '/',
  port: dfltPort,
  getDefaultModules,
  postcss: () => {
    return [
      require('autoprefixer')({
        browsers: ['last 2 versions', 'ie >= 8'],
      }),
    ];
  },
  resolve: {
    root: path.resolve('src'),
    modulesDirectories: ['node_modules'],
  },
  currentEnv: CURRENT_ENV,
  plugins: [
    // HappyPack: 为了让打包任务多线程并列运行
    new HappyPack({
      id: 'js',
      threadPool: happyThreadPool,
      loaders: [
        'ng-annotate',
        'babel?presets[]=es2015&plugins[]=transform-runtime&cacheDirectory=true',
      ],
    }),
    new HappyPack({
      id: 'html',
      threadPool: happyThreadPool,
      loaders: ['raw!html-minify'],
    }),
    new HappyPack({
      id: 'scss',
      threadPool: happyThreadPool,
      loaders: ['style-loader!css-loader!postcss-loader!sass-loader?outputStyle=expanded'],
    }),
    new webpack.ProvidePlugin({
      $: 'jquery',
      jQuery: 'jquery',
    }),
    new webpack.NoErrorsPlugin(), // 允许错误不打断程序
    new webpack.DefinePlugin({
      'process.env': CURRENT_ENV,
    }),
    new BowerWebpackPlugin({
      searchResolveModulesDirectories: false,
    }),
  ],
};
```



##### base.js

基础配置框架，接下来 dev 和 dist 会在这之上进行配置覆盖。

```javascript
const path = require('path');
const defaultSettings = require('./defaults');

module.exports = {
  port: defaultSettings.port,
  debug: true,
  output: {
    path: path.join(__dirname, '/../dist/ui'),
    filename: '[name].js',
    publicPath: `.${defaultSettings.publicPath}`,
  },
  // webpack-dev-server 的配置
  devServer: {
    contentBase: './src',
    historyApiFallback: true, // enables support for history API fallback
    inline: true,
    hot: true,
    port: defaultSettings.port,
    // noInfo: true, // suppress boring information
    stats: {
      colors: true, // add some colors to the output
    },
  },
  resolve: defaultSettings.resolve,
  module: defaultSettings.getDefaultModules(),
  postcss: defaultSettings.postcss,
};
```



##### dev.js

开发环境配置

```javascript
const webpack = require('webpack');
const baseConfig = require('./base');
const defaultSettings = require('./defaults');

const config = Object.assign({}, baseConfig, {
  watch: true,
  entry: {
    app: [
      `webpack-dev-server/client?http://localhost:${defaultSettings.port}`, // 这一行很重要，使 webpack-dev-server 使用 inline 模式，iframe 模式下 url 不会变比较头疼
      'webpack/hot/dev-server',
      './src/core/bootstrap.js',
    ],
  },
  cache: true,
  devtool: 'cheap-source-map',
  plugins: defaultSettings.plugins.concat([
    new webpack.HotModuleReplacementPlugin(), // 热替换
    new webpack.DllReferencePlugin({
      context: __dirname,
      manifest: require('../vendor-manifest.json'),
    }),
  ]),
});

module.exports = config;
```



##### dist.js

生产环境配置

```javascript
const webpack = require('webpack');
const baseConfig = require('./base');
const defaultSettings = require('./defaults');

const config = Object.assign({}, baseConfig, {
  entry: {
    app: ['./src/core/bootstrap.js'],
  },
  cache: false,
  plugins: defaultSettings.plugins.concat([
    new webpack.optimize.DedupePlugin(), // 查找相等或近似的模块，避免在最终生成的文件中出现重复的模块
    new webpack.optimize.UglifyJsPlugin({ // js 压缩
      compress: {
        warnings: false,
      },
    }),
    new webpack.optimize.OccurenceOrderPlugin(), // 给经常使用的模块分配最小长度的id
    new webpack.optimize.AggressiveMergingPlugin(),
    new webpack.BannerPlugin('Q29weXJpZ2h0IMKpIDIwMTYgRGFvQ2xvdWQgQWxsIHJpZ2h0cyByZXNlcnZlZC4g'), // 给输出的文件头部添加注释信息
  ]),
});

module.exports = config;
```



#### webpack-dev-server

webpack-dev-server 是一个小型的 node.js Express 服务器，使用它，可以为webpack 打包生成的资源文件提供 Web 服务。

##### 两种模式

webpack-dev-server 有两种模式支持自动刷新——iframe 模式和 inline 模式。

* iframe 模式下：页面是嵌套在一个 iframe 下的，在代码发生改动的时候，这个 iframe 会重新加载，切换页面 url 不会变化。


* inline 模式下：一个小型的 webpack-dev-server 客户端会作为入口文件打包，这个客户端会在后端代码改变的时候刷新页面。切换页面 url 会变化。

使用 iframe 模式无需额外的配置，只需在浏览器输入类似 http://localhost:8012/webpack-dev-server/ 这样的 url。

使用inline模式有两种方式：命令行方式和Node.js API。

1) 命令行方式比较简单，只需加入--line选项即可。

例如：`webpack-dev-server —inline`

使用 —inline 选项会自动把 webpack-dev-server 客户端加到 webpack 的入口文件配置中。

**注意：**使用 webpack-dev-server 命令行的时候，会自动查找名为webpack.config.js 的配置文件。如果你的配置文件名称不是 webpack.config.js，需要在命令行中指明配置文件。例如，配置文件是webpack.config.dev.js：`webpack-dev-server --inline --config webpack.config.dev.js`。

2) Node.js API方式需要手动把 `webpack-dev-server/client?http://localhost:8080` 加到配置文件的入口文件配置处，因为 webpack-dev-server 没有 `inline:true` 这个配置项。

##### 热替换

webpac-dev-server 支持 Hot Module Replacement，即模块热替换，在前端代码变动的时候无需整个刷新页面，只把变化的部分替换掉。使用HMR功能也有两种方式：命令行方式和 Node.js API。

* 命令行方式同样比较简单，只需加入 --line —hot 选项。—hot 这个选项干了一件事情，它把 `webpack/hot/dev-server` 入口点加入到了 webpack 配置文件中。这时访问浏览器，你会看见控制台的log信息：

  ```
  [HMR] Waiting for update signal from WDS...
  [WDS] Hot Module Replacement enabled.
  ```

  HMR前缀的信息由 `webpack/hot/dev-server` 模块产生，WDS前缀的信息由 `webpack-dev-server` 客户端产生。

* Node.js API方式需要做三个配置：

​	1) 把 `webpack/hot/dev-server` 加入到 webpack 配置文件的 entry 项；

​	2) 把 `new webpack.HotModuleReplacementPlugin()` 加入到webpack配置文			件的 plugins 项；

​	3) 把 `hot:true` 加入到 webpack-dev-server 的配置项里面。



**注意：**要使 HMR 功能生效，还需要做一件事情，就是要在应用热替换的模块或者根模块里面加入允许热替换的代码。否则，热替换不会生效，还是会重刷整个页面。

```javascript
if(module.hot) module.hot.accept();
```

遗憾的是，在我的项目中，由于 controller 或 service 这些在 angular 中注册了的模块都是定义一个 class 来写的，热替换并不能完全成功，而 class 之外都是可以的，貌似和实例化有关，具体原因并没有去深究。所以现在仍然是刷新整个页面。



#### 性能优化

性能优化很重要，优化前后确实有很大差距，我是参考这篇文章的：[如何十倍提升你的 webpack 构建效率](http://gold.xitu.io/entry/5768e6ab207703006b310f95)



#### 其它参考文章

http://www.jianshu.com/p/941bfaf13be1

http://shmck.herokuapp.com/webpack-angular-part-1/

http://engineering.invisionapp.com/post/optimizing-webpack/