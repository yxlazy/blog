---
title: react-router 简单介绍与本地调试环境搭建
date: 2021-09-29 23:23:01
tags: [react, react-router, 源码解析]
category: react-router
description: react-router是react标配的路由组件，现在为了能更好的研究它，我们先搭建一个本地调试环境用于后续的研究吧
---

## 前言

`react-router`是`react`的浏览器路由组件，使用`react-router`可以轻松的实现浏览器路由跳转。`react-router`目前已经更新到`v5.3.0`, 最新的`history`也已经更新到`v5.0.1`。目前，从`Github`仓库可以看出，官方人员正在全力使用`TypeScript`重构两个项目, 新版的`history`仓库和原来的仓库并不兼容。我们从`npm`上直接添加的`history`仍然是`v4`版本。

> 使用`TypeScript`重构的两个仓库: 
> [react-router仓库](https://github.com/remix-run/react-router/tree/dev)
> [history仓库](https://github.com/remix-run/history/tree/main)

> 以下介绍的项目版本：`react-router`是`v5.3.0`, `history`是`v4.10.1` 

## react-router

`react-router`仓库, 分为四个包: 

- `react-router`: `react router`的核心组件
- `react-router-dom`: 浏览器路由组件
- `react-router-native`: `react native`的路由组件
- `react-router-config`: 静态路由组件（如果需要使用`react router`的静态路由，需要引用这个包，重构后这个包，已经被删除）

在使用过程中，我们只需要添加对应的包即可，不需要添加`react-router`，例如，在浏览器中使用添加`react-router-dom`；在`native`中使用添加`react-router-native`；如果需要使用以前的静态路由，就添加`react-router-config` 。

## 本地调试环境搭建

在阅读源码过程中难免会有令人头痛的代码片段部分，而这部分，想要弄明白，最好的方式就是通过调试，但怎么在本地调试代码呢？我们可以借助`webpack`的`resolve.alias`进行本地代码映射：

```js
resolve: {
  extensions: ['.js', '.jsx'],
  alias: {
        'react-router-dom':path.resolve(__dirname, '../src/react-router/packages/react-router-dom/modules'),
        'react-router': path.resolve(__dirname, '../src/react-router/packages/react-router/modules'),
        'history': path.resolve(__dirname, '../src/history/modules')
  }
}
```

除此之外，`react-router`还需要其他包，因此需要在`react-router`和`history`下运行`yarn`安装对应的包。



然后，就可以引入到项目中了：

```jsx
import { BrowserRouter as Router, Switch, Route } from "react-router-dom";
function Start() {
  return (
    <div className="App">
      <p>test page</p>
    </div>
  );
}
const Root = () => {
  return(
    <div>
      <p>root page</p>
    </div>
  )
}

function App() {
  return(
    <Router>
      <Switch>
        <Route path='/test' component={Start}/>
        <Route component={Root} />
      </Switch>
    </Router>
  )
}

export default App;
```

