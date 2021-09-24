---
title: Context
date: 2021-09-23 22:40:26
tags: [react, javascript]
category: react
description: React为我们提供了Context用于将数据进行广播，消费组件只需使用特定的方法就能拿到广播的数据，不用再采用层层递传的方式...
---

![](React之Context/image-20210924142001161.png)

------



## 前言

`React`为我们提供了`Context`用于将数据进行广播，消费者只需使用特定的方法就能拿到广播的数据，不用再采用层层递传`props`的方式。如果组件之间需要共享数据，也可以采用`Context`的方式，如`react-router`中的`Router`组件：

```react
...
render() {
    return (
      <RouterContext.Provider
        value={{
          history: this.props.history,
          location: this.state.location,
          match: Router.computeRootMatch(this.state.location.pathname),
          staticContext: this.props.staticContext
        }}
      >
        <HistoryContext.Provider
          children={this.props.children || null}
          value={this.props.history}
        />
      </RouterContext.Provider>
    );
  }
...
```

使用`Provider`包裹组件后，后续就能直接从`context`中获取最新的`history`等数据。

> 在什么情况下需要使用`Context` : *https://react.docschina.org/docs/context.html#when-to-use-context* 
>
> 使用之前的考虑: *https://react.docschina.org/docs/context.html#before-you-use-context* 

## React.createContext

用于创建一个`Context`对象，其中包含`Provider`和`Consumer`两个组件。`Provider`组件可以提供用于`Consumer`组件消费的`context`。同时可以在创建`Context`对象时，为其指定一个默认值：

```react
const NewContext = React.createContext(defaultValue);
```



如果`Consumer`组件不能在所处的树中匹配到`Provider`，将会使用这个`defaultValue`。

> 如果直接给`Provider`的`value`赋值为`undefined`，消费组件的`defaultValue`不会生效

## Context.Provider

`Provider`可以为消费组件提供`context`，通过将要广播的数据放入`value`，内部的消费组件就可以消费这个`value`:

```react
const {Provider} = NewContext;
...
<Provider value='provider'>
	<Button />
</Provider>
...
```

`Provider`可以进行嵌套，但`Consumer`只会消费**最近**的`Provider`。

如果`Provider`的`value`发生变化，**它内部的所有的消费组件都会重新渲染**，即使使用了`shouldComponentUpdate`进行了优化处理，也不会影像消费组件的重新渲染。

> 如果为`Provider`提供的是对象，由于该父组件进行重渲染时，每次`value`都会得到一个新的对象，从而consumers组件中也会发生不必要的渲染。

解决方式是将该对象保存在`state`中:

```react
const App = () => {
  const [state, setState] = useState({});
  useEffect(() => {
    ...
    setState({theme: '#fff'})
    ...
  })
  return(
  <NewContext.Provider value={state}>
  	<Button />  
  </NewContext.Provider>)
}
```

## Class.contextType

通过给组件的`contextType`属性赋值为`Context`对象后，可以在该组件的内部拿到`context`: 

```react
class App extends React.Component {
  ...
  componentDidMount() {
    ...
    let value = this.context
    ...
  }
  ...
  render() {
    ...
    let value = this.context
    ...
  }
}
App.contextType = NewContext
```

可以在组件的任何生命周期中拿到`context`。也可以使用`static`关键字进行初始化:

```react
class App extends React.Component {
  static contextType = NewContext
	render() {
    ...
    let value = this.context
    ...
  }
}
```



## Context.Consumer

`contextType`只适用于类组件，对于函数组件，可以使用`Consutmer`组件获取`context`: 

```react
const App = () => {
  return(
  	<NewContext.Consumer>
    	{context => {
        // 拿到context，并进行其他逻辑
        
        // 返回React节点
        return <Button style={{background: context.theme}}/>
      }}
    </NewContext.Consumer>
  )
}
```

`Consumer`组件中需要一个函数来接收当前的`context`，函数返回一个`React`节点。



## Context.displayName

`Context`对象接收一个`displayName`属性，类型为字符串，React DevTools 使用该字符串来确定 `context` 要显示的内容: 

```react
const MyContext = React.createContext(/* some value */);
MyContext.displayName = 'MyDisplayName';

<MyContext.Provider> // "MyDisplayName.Provider" 在 DevTools 中
<MyContext.Consumer> // "MyDisplayName.Consumer" 在 DevTools 中
```



## useContext

`useContext`是`React`提供的一个`hooks API`，可以用来拿到对应的`Context`对象提供的`context`：

```react
// context.js
export const NewContext = React.createContext(defaultValue);

// app.js
import {NewContext} from './context'
import {Buttons} from './buttons'
const App = () => {
  ...
  return(
  	<NewContext.Provider value={{theme: '#fff'}}>
    	...
      <Buttons />
    </NewContext.Provider>
  )
}

// buttons.js
import {Button} from './button'
const Buttons = () => {
  ...
  return(
    ...
  	<Button />
    ...
  )
}

// button.js
import {useContext} from 'react'
import {NewContext} from './context'
const Button = () => {
  const context = useContext(NewContext)
  
  return (
  	<button style={{background: context.theme}}>按钮</button>
  )
}
```

当组件上层最近的 `<NewContext.Provider>` 更新时，该 Hook 会触发重渲染，并使用最新传递给 `NewContext` provider 的 context `value` 值。即使祖先使用 [`React.memo`](https://react.docschina.org/docs/react-api.html#reactmemo) 或 [`shouldComponentUpdate`](https://react.docschina.org/docs/react-component.html#shouldcomponentupdate)，也会在组件本身使用 `useContext` 时重新渲染。

## 参考

[1] *https://react.docschina.org/docs/context.html*

[2] *https://react.docschina.org/docs/hooks-reference.html#usecontext* 

