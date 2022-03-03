---
title: react-router v6 api学习
date: 2022-02-15 23:53:52
tags: [react, react-router]
category: react-router
description: react router默认版本已经是v6版本了，你还不准备学习？
---
## 概述
react router目前已经更新到v6版本，该版本不仅使用`TypeScript`进行了重构，而且还新增了很多的功能，其与v5版本可以说已完全没有关联。
现在v6已成为默认安装版本。

react router v6分为3个包：
- react-router: 包含核心功能
- react-router-dom: web端使用的包
- react-router-native: rn端使用的包
> react-router-dom和react-router-native已经自动引入了react-router，所以我们在开发时，只需要安装对应开发环境的包即可

## 引用路由组件
对于web端，支持两种形式的路由:
- `<HashRouter />`: hash路由，当路由发生改变不会重新向服务端发起请求
- `<BrowserRouter />`: 使用浏览器内置历史堆栈进行导航
  
> 新的组件已不再需要显示添加history，其已经在组件内部实现

```js
// 使用hash
import {HashRouter, Routes, Route} from 'react-router-dom'

function App() {
  return(
    <HashRouter>
      <Routes>
        <Route path='/' element={<Home />}>
        <Route path='/about' element={<About />}>
      </Routes>
    </HashRouter>
  )
}

// 不使用hash
import {BrowserRouter, Routes} from 'react-router-dom'

function App() {
  return(
    <BrowserRouter>
      <Routes>
        <Route path='/' element={<Home />}>
        <Route path='/about' element={<About />}>
      </Routes>
    </BrowserRouter>
  )
}
```

> 注意，如果不使用hash，则每个路由必须要有一个与之对应的服务端路由，否则会出现404

`<NativeRouter />`: React Native端使用的路由

`<MemoryRouter />`: 其将路由存储在一个数组中，不依赖外部源，可以适合各种场景，例如测试环境下


### `<Routes />`
检查与之匹配的path，并渲染对应的`<Route />`

### `<Route />`
渲染匹配path的React Element


## 路由跳转
### `<Link />`
其内部会渲染一个a标签，点击会跳转到对应的路由
```js
<Link to='/home'>跳转到home路径</Link>
```

### `<NavLink />`
是一个特殊的`<Link />`组件，常用于渲染导航栏选中时的高亮
```ts
<NavLink style={({isActive: boolean}) => isActive ? 'blue' : 'white'}>
  Home
</NavLink>
```

### `<Navigate />`
对`useNavigate()`的封装
```js
<Navigate to='/home' />
```
> 这个组件只是为了在Class组件写法中使用，如果是使用Function组件,建议使用hooks写法

### `useNavigate()`: 
改变路由：
```js
const navigate = useNavigate();

// 跳转到/login路由
navigate('/login');

// 默认跳转类型为PUSH, 可以设置跳转的类型为REPLACE
navigate('/login', {replace: true});

// 可以直接传递数值跳转到指定某个记录，与window.history.go()类似
navigate(-1);
```

## 使用嵌套路由
### `<Outlet />`
实现嵌套路由的渲染
```js
ReactDom.render(
  <Routes>
    <Route path='/' element={<App />}> 
      <Route path='/home' element={<Home />} />
    </Route>
  </Routes>,
  document.getElementById('root');
);
// App.js
function App() {
  return(
    <div>
      <p>渲染App组件<p>
      {/**如果不添加这个，当path匹配/home时，页面将不会渲染Home组件**/}
      <Outlet />
    </div>
  );
}
```

### `useOutlet()`
返回路由层次结构中此级别的子路由的元素, `<Outlet />`的hooks版
```js
// App.js
function App() {
  const outlet = useOutlet();
  return(
    <div>
      <p>渲染App组件<p>
      {/**如果不添加这个，当path匹配/home时，页面将不会渲染Home组件**/}
      {outlet}
    </div>
  );
}
```

### `useOutLetContext()`
用来向子组件传递属性
```js
// App.js
function App() {
  const [message, setMessage] = useState("");
  return (
    <div>
      <p>渲染App组件<p>
      {/**通过context向子组件广播属性**/}
      <Outlet context={[message, setMessage]} />
    </div>
  );
}

// Home.js
function Home() {
  // 子组件通过useOutletContext() 拿到父组件传递的属性
  const [message, setMessage] = useOutletContext();
  return (
    <div>
      home
      <p>{message}</p>
      <input onChange={(e) => setMessage(e.target.value)} />
    </div>
  );
}
```


## Index Routes 概念

什么叫"index route"?
- index route 渲染在父路由的outlet
- 当一个父路由被匹配时，其他子路由都未被匹配，将会匹配index route，
- index route 是父路由的默认子路由
- 如果用户尚未点击导航的任何一个路由时，index route将会被渲染

> 可能我理解的不是很准确，可以查看[官方的解释](https://reactrouter.com/docs/en/v6/getting-started/tutorial#index-routes)



```js
ReactDom.render(
  <Routes>
    <Route path='/' element={<App />}> 
      <Route path='/home' element={<Home />} />
      <Route path='/about' element={<About />}>
    </Route>
  </Routes>,
  document.getElementById('root');
);
// App.js
function App() {
  return(
    <div>
      <p>渲染App组件<p>
      {/**如果不添加这个，当path匹配/home时，页面将不会渲染Home组件**/}
      <Outlet />
    </div>
  );
}
```

对于上面的例子，当path为'/'时，此时的渲染组件为：
```js
<App />
```

如果添加一个Index Route路由：
```js
ReactDom.render(
  <Routes>
    <Route path='/' element={<App />}> 
      {/** Index Route **/}
      <Route index element={<Home />} />
      <Route path='/about' element={<About />}>
    </Route>
  </Routes>,
  document.getElementById('root');
);
```
当path='/'时，此时渲染的组件为：
```js
<App>
  <Home />
</App>
```

## Not Found Routes 概念
当没有路由能够匹配，我们常常会添加一个NotFound页面:
```js
<Routes>
  <Route path='/' element={<Home />}>
  <Route path='*' element={<NotFound />}>
</Routes>
```
`path='*'`将匹配所有的URL,但优先级最低，只有不存在匹配的路由时才会匹配它（尝试了下，这个匹配完全不依赖放置的顺序，即你可以任意放置`path='*'`路由的位置🤔感觉比以前好用了，心智模式至少得到了降低）

## hooks
这里介绍前面没有介绍的hook

### useHref()
传入一个`To`类型的数据，返回一个拼好的URL, 可以用来放在属性`to`的位置
```ts
  // type To = Partial<Location> | string;

  const to: To = {
    pathname: "/123456",
    search: "a=b&c=d",
    hash: "home"
  };
  // const href = useHref('/home?a=b&c=d');
  const href = useHref(to);

  <Link to={href} />
```

### useLinkClickHandler()
返回一个点击事件回调，通常用于自定义`<Link />`组件
```js
  const handleClick = useLinkClickHandler(to);

  <div onClick={handleClick}>点我跳转</div>
```

### useLinkPressHandler()
同useLinkClickHandler(), 这是用于rn端的

### useInRouterContext()
用于判断组件是否在`<Router />`中渲染, true为是，false为否
```js
  const isInRouterRender = useInRouterContext();

  <div>
    {isInRouterRender ? '组件渲染在<Router />中' : '组件可能飞了'}
  </div>
```

### useNavigationType()
返回进入当前页面的类型或当前导航类型
```ts
type NavigationType = "POP" | "PUSH" | "REPLACE";
```
> 页面初次加载是NavigationType为'POP'类型, 后面如果不设置，默认是'PUSH'类型

### useMatch()
返回给定路径上相对于当前位置的路由的匹配数据。
```ts
  interface PathMatch<ParamKey extends string = string> {
    params: Params<ParamKey>;
    pathname: string;
    pattern: PathPattern;
  }

  interface PathPattern {
    path: string;
    caseSensitive?: boolean;
    end?: boolean;
  }

  const match: PathMatch<ParamKey> | null = useMatch('/home');
```

### useParams()
返回当前URL中的params，并组成一个键/值对对象
```js
import * as React from 'react';
import { Routes, Route, useParams } from 'react-router-dom';

function ProfilePage() {
  // Get the userId param from the URL.
  let { userId } = useParams();
  // ...
}

function App() {
  return (
    <Routes>
      <Route path="users">
        <Route path=":userId" element={<ProfilePage />} />
        <Route path="me" element={...} />
      </Route>
    </Routes>
  );
}
```

### useResolvedPath()
返回一个解析后的Path对象
```ts
declare function useResolvedPath(to: To): Path;

interface Path {
  pathname: string;
  search: string;
  hash: string;
}
```

### useRoutes()
等同于`<Routes />`，接收一个和`<Route />`一样参数的对象组成的数组，还是看例子吧：
```js
  // 我们将上面的ReactDOM.render渲染用hooks重写
function Root() {
  const element = useRoutes([
    {
      path: '/',
      element: <App />,
      children: [
        {
          path: '/home',
          element: <Home />
        },
        {
          path: '/about',
          element: <About />
        }
      ]
    },
    // 添加一个新的路由
    { path: "team", element: <AboutPage /> }
  ]);

  return element;
}

ReactDOM.render(<Root />, document.getElementById('root'));
```

### useSearchParams()
用来操作URL的search参数，从此我们不在需要自己去写一个解析search的工具函数了。使用方式与React.useState一样：
```js
// http://localhost?id=1
const [searchParams, setSearchParams] = useSearchParams();

// 获得id字段得值
searchParams.get('id');

// 获取所有的id字段值，返回数组
searchParams.getAll('id'); // => ['1']

// 更改id字段得值
setSearchParams({id: 2});
```
searchParams是一个URLSearchParams类型实例, setSearchParams其效果与navigate（useNavigate()的返回值）类似，只是前者只改变url的search段
> 关于URLSearchParams类型实例，详情查看 https://developer.mozilla.org/zh-CN/docs/Web/API/URLSearchParams
> 注意URLSearchParams的兼容性

### useLocation()
返回当前所在页面的location对象（和window.location还是有区别的）。可以用来处理一些当前页面的location发生改变时的副作用

> 有些api并没有在这里提及
> - [`<Router>`](https://reactrouter.com/docs/en/v6/api#router)
> - [`<StaticRouter>`](https://reactrouter.com/docs/en/v6/api#staticrouter)
> - [createRoutesFromChildren](https://reactrouter.com/docs/en/v6/api#createroutesfromchildren)
> - [generatePath](https://reactrouter.com/docs/en/v6/api#generatepath)
> - [Location](https://reactrouter.com/docs/en/v6/api#location)
> - [matchRoutes](https://reactrouter.com/docs/en/v6/api#matchroutes)
> - [renderMatches](https://reactrouter.com/docs/en/v6/api#rendermatches)
> - [matchPath](https://reactrouter.com/docs/en/v6/api#matchpath)
> - [resolvePath](https://reactrouter.com/docs/en/v6/api#resolvepath)
> - [createSearchParams](https://reactrouter.com/docs/en/v6/api#createsearchparams)


## 参考
[1] *react router官网 https://reactrouter.com/docs/en/v6/api*
