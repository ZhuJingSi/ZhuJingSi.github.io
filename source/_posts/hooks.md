---
title: React Hooks
date: 2019-03-05 18:37:18
tags: 
  - react
  - hooks
categories: 
  - 前端
---

*Hooks 在 React v16.7.0-alpha 及以上版本中可用，可以在函数组件中使用类似类函数中的 state(状态) 和其他 React 功能。具体的功能和用法在 官方文档 中有非常详尽的介绍，这里主要是提炼一些我认为值得注意的点以及表述一些自己的思考。*

### 什么是 Hooks

**Hooks** 是一个 React 函数组件内一类特殊的函数（通常以 “use” 开头，比如 “useState”），使开发者能够在函数组件里使用 state 和 life-cycles，以及使用自定义 hook 复用业务逻辑。

<!-- more -->

### 动机

原先的 React 拥有两种创建组件的方式：**函数组件** 和 **类组件**。

> 函数组件就是一个传入参数返回 React 元素的 JS 函数，它没有内部 state 和生命周期，是无状态函数式组件，通常作为没有复杂交互的纯展示组件。
>
> 类组件顾名思义就是用一个 [ES6 的 class](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Classes) 来定义的组件，它拥有 state 和完善的生命周期。

日常开发中我们可能更多的会使用类组件，因为很多情况下都不可避免地要使用 state 和生命周期。然而当使用场景越来越复杂的情况下，类组件却暴露出一些问题：

#### 在组件之间重用有状态逻辑很困难

在以往，如果想要复用一段逻辑（比如：在所有页面所有组件都需要获取当前用户的身份权限，以此做一些特殊处理时），通常会使用：1. 带有这段复用逻辑的 [高阶组件](http://react.html.cn/docs/higher-order-components.html) 2. 带有这段复用逻辑的普通组件配合上 [render props(渲染属性)](http://react.html.cn/docs/render-props.html)。但这导致的问题就是当我们的应用规模变得越来越大的时候，一些无关 UI 的 wrapper 组件越来越多，React 组件树变得越来越臃肿（在 devtool 中可以甚至看到数十层 wrapper），使开发和调试的效率变得很低。

#### 按生命周期方法划分逻辑将变得难以理解

每个生命周期方法通常包含一组不相关的逻辑。例如，组件可能在 `componentDidMount` 和 `componentDidUpdate`中执行一些数据获取。然而，相同的 `componentDidMount` 方法可能还包含一些不相关的逻辑，它们设置事件监听器，并在 `componentWillUnmount` 中执行清理。本应一起更改的相互关联的代码会被分离，完全不相关的代码最终会组合在一个方法中。这很容易引入错误和不一致，比如常常被遗忘的“取消订阅”和“取消轮询”等逻辑。

#### 性能优化困难

相比于函数组件，类组件很难获得提前编译带来的好处，同时类不能很好地压缩，并且它们使得热更新加载变得片状和不可靠。

鉴于以上类函数的种种短板，React 官方顺理成章地进一步拥抱函数式编程思想，于是设计出 Hooks 这样的特殊函数来弥补函数组件相对于类组件缺少的 state 和生命周期功能，并打算让 Hooks 涵盖所有现有的类用例，希望在未来大家更多地去用函数组件而非类组件。

### 规则

1. Hooks 命名都以 “use” 开头
2. Hooks 只能在 React 函数式组件中或另一个自定义 Hooks 中调用
3. Hooks 只能在函数顶层调用，不能在循环、条件或嵌套函数中调用（因为 React 依赖于调用 Hooks 的顺序，在条件语句中可能会打乱顺序）

安装 [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks) 插件可以强制执行这些规则。

### 用法

这里简单说一下几个比较核心的：

#### useState

相当于类函数中的一套 state 管理方法，返回长度为 2 的一个数组，包含： stateful(有状态) 值，以及更新这个状态值的函数。

```javascript
const [state, setState] = useState(initialState);
```

可以看出来，这种写法类比到类组件中就相当于 this.state 和 this.setState，只不过这里每次只能处理一个 state。有同学可能会说这样好麻烦，state 如果很多的话写 useState 都要写一长串。但这种写法其实很方便我们将 state 归类处理，即将同类的 state 合并到一个对象中作为 `initialState`，逻辑更清晰，也方便未来将一些相关逻辑提取到自定义 Hook 中。

```javascript
const [position, setPosition] = useState({ left: 0, top: 0 });
const [size, setSize] = useState({ width: 100, height: 100 });
```

> 另外，`setState` 和 this.setState 一样，同步写法前面的赋值会被最后一个覆盖，可以看 [我之前整理的规则](https://cisy.me/setState2/)，`setState`也同样能接受一个函数。

#### useEffect

相当于类函数中的生命周期函数的作用，不同的是，传递给 `useEffect` 的函数会在 layout(布局) 和 paint(绘制) 后触发。这使得它适用于许多常见的 side effects ，例如设置订阅和事件处理程序，因为大多数类型的工作不应阻止浏览器更新屏幕。

然而当你需要在浏览器绘制之前同步触发一些事件，就需要用到 [`useLayoutEffect`](http://react.html.cn/docs/hooks-reference.html#uselayouteffect)，它和 useEffect 除了触发时间之外没什么不同。

`useEffect` 接收一个函数，该函数可选地返回一个函数（React 将在清理时运行它，常用于取消订阅等逻辑）。

```javascript
useEffect(
  () => {
    const subscription = props.source.subscribe();
    return () => {
      subscription.unsubscribe();
    };
  },
  [props.source],
);
```

值得注意的是，`useEffect` 会在每次组件 rerender 时都触发一次，如果希望在特定条件下触发 effects，就给 `useEffect` 传入第二个参数： effect 所依赖的值数组，其中任何一个值发生了变化就会触发 `useEffect` 的第一个函数参数，否则就不会有反应，So 如果传一个 `[]` 则只有在组件初始化和卸载时才会执行了。

```
第一次渲染：subscribe
第二次渲染：unsubscribe、subscribe // 没有依赖或依赖数组有变化时触发
第三次渲染：unsubscribe、subscribe // 没有依赖或依赖数组有变化时触发
组件卸载：  unsubscribe
```

#### Custom hooks

官方已经内置了一些 hooks，同时为了解决动机中的复用状态逻辑的问题，就需要自定义 Hooks 了。

当我们想要在两个 JS 函数之间共享逻辑时，我们会将共享逻辑提取到第三个函数。自定义 Hooks 就是多个组件之间可共享的状态逻辑的提取函数，但是每次使用自定义 Hook 时，它内部的所有状态和效果都是完全隔离的。

具体用法可以看 [官方文档](http://react.html.cn/docs/hooks-custom.html)，在此不赘述。

#### useMemo

性能是永恒的关注点，纯函数组件在父组件重绘的时候是会无条件跟着重绘的，这着实是个很大的性能隐患。React 团队当然不能让这个问题阻碍到用户迁移到函数式写法，useMemo 和 memo 可以从父组件和子组件的视角帮助函数组件实现 `shouldComponentUpdate`。

##### 在父组件控制子组件更新

在父组件内根据父组件内部状态决定子组件是否更新的话可以用到 useMemo 这个 Hooks：

```javascript
function Parent({ a, b }) {
  // Only re-rendered if `a` changes:
  const child1 = useMemo(() => <Child1 a={a} />, [a]);
  // Only re-rendered if `b` changes:
  const child2 = useMemo(() => <Child2 b={b} />, [b]);
  return (
    <>
      {child1}
      {child2}
    </>
  )
}
```

##### 子组件自己控制

```javascript
const Button = React.memo((props) => {
  // 只有当 props 变化的时候才会执行
});
```

用 `React.memo` 包裹你的组件就可以了，是否 rerender 取决于浅比较 props 有没有变化，相当于 `PureComponent`。

还可以添加第二个参数，比较新旧 props 的自定义函数，如果返回 `true` ，则跳过更新。

```javascript
const Button = React.memo((props) => {
  // 只有当 props 变化的时候才会执行
}, (prevProps, nextProps) => {
  // 返回 false 正常渲染
  // 返回 true 则跳过渲染
});
```

#### 其它 API

其它 API 翻文档就好了。

- Basic Hooks
  - [`useState`](http://react.html.cn/docs/hooks-reference.html#usestate)
  - [`useEffect`](http://react.html.cn/docs/hooks-reference.html#useeffect)
  - [`useContext`](http://react.html.cn/docs/hooks-reference.html#usecontext)
- Additional Hooks
  - [`useReducer`](http://react.html.cn/docs/hooks-reference.html#usereducer)
  - [`useCallback`](http://react.html.cn/docs/hooks-reference.html#usecallback)
  - [`useMemo`](http://react.html.cn/docs/hooks-reference.html#usememo)
  - [`useRef`](http://react.html.cn/docs/hooks-reference.html#useref)
  - [`useImperativeMethods`](http://react.html.cn/docs/hooks-reference.html#useimperativemethods)
  - [`useLayoutEffect`](http://react.html.cn/docs/hooks-reference.html#uselayouteffect)

### 总结

React **没有从 React 中移除类的计划**，Hooks 时完全向后兼容的，你可以慢慢地在合适的组件尝试或做迁移。

可以明显地看到，React 越来越趋向于向函数式编程靠拢，未来的前端优化方向也大多基于此。