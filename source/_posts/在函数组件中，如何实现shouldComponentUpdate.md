---
title: 在函数组件中，如何实现shouldComponentUpdate
date: 2022-08-04 17:30:49
tags: [react, javascript]
category: react
description: 在函数组件中，我们该如何实现`class`组件才有的`shouldComponentUpdate()`生命周期呢？
---

在函数组件中，我们该如何实现`class`组件才有的`shouldComponentUpdate()`生命周期呢？

首先我们必须知道`shouldComponentUpdate()`的作用是什么，官方文档解释的是，**当`props`和`state`发生改变时，`shouldComponentUpdate()`会在渲染之前被调用，默认的返回值是`true`，当返回值为`false`时，将会阻止本次渲染**

既然已经知道了它的作用，那我们在函数组件中该如何实现呢？

在函数组件中，要阻止组件重新渲染就需要使用到`React.memo()`，`React.memo`是类似于高阶组件，但只适用于函数组件，我们在第二个参数中，通过返回`true`来阻止函数组件的渲染。

可以复制下面的代码，运行查看效果

```javascript
import { Component, useState, memo } from "react";

const Child = (props) => {
  return <div>{props.type}组件渲染结果：{props.text}</div>;
};

// 函数组件
const ContentFn = memo((props) => {
  return <Child type='函数' {...props} />;
}, (prevProps, nextProps) => {
  // 返回true，组织渲染
  return nextProps.disabled
});

// 类组件
class ContentClass extends Component {
  constructor(props) {
    super(props);
  }

  shouldComponentUpdate(nextProps, nextState) {
    // 阻止渲染
    return !nextProps.disabled;
  }

  render() {
    return <Child type='类' {...this.props} />;
  }
}

const App = () => {
  const [text, setText] = useState("");
  const [disabled, setDisabled] = useState(false);
  return (
    <div>
      <div>
        <input onChange={(e) => setText(e.target.value)} />
        <button onClick={() => setDisabled((disabled) => !disabled)}>
          {disabled ? "正常渲染" : "阻止渲染"}
        </button>
      </div>
      <ContentFn text={text} disabled={disabled} />
      <ContentClass text={text} disabled={disabled} />
    </div>
  );
};

export default App;

```

特别说明，`shouldComponentUpdate()`和`memo()`两个的返回值相反，`shouldComponentUpdate()`返回`false`会跳过渲染，而`React.memo()`是在返回`true`的情况下才跳过渲染。

事实上，`React.memo()` 并不是阻断渲染，而是跳过渲染组件的操作并直接**复用最近一次渲染**的结果，这与 `shouldComponentUpdate()` 是完全不同的。

而对于这两个函数，则都是用于性能优化的手段，在类组件中，官方提供了`PrueComponent`类组件实现了`shouldComponentUpdate()`，在函数组件中，`React.memo()`通常需要搭配`useCallback()`和`useMemo()`才能更好的发挥其作用。

