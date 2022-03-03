---
title: axios源码学习
date: 2022-03-01 23:33:53
tags: [源码解析]
category: other
description: axios是一个轻量无依赖的Promise版的http请求库，不仅可以在浏览器端使用，也可以在node端使用，今天我们就来学习一下这款备受欢迎的库。
---

`axios`是一个轻量无依赖的`Promise`版的`http`请求库，不仅可以在浏览器端使用，也可以在node端使用，今天我们就来学习一下这款备受欢迎的库。

在使用`axios`时，我们可以通过以下方式发起一个`axios`请求
```js
axios({})
new axios.Axios({})
axios.create({})
axios['get']()
```
其实最后都是调用的`Axios.prototype.request`方法，我们可以看一下`createInstance()`这个函数到底做了什么，可以看到，这个函数主要就是用户创建一个`axios`对象：
```js
function createInstance(defaultConfig) {
  // 创建一个axios对象
  var context = new Axios(defaultConfig);
  // 绑定Axios.prototype.request中的this并创建一个instance变量
  var instance = bind(Axios.prototype.request, context);
  // 将Axios原型对象扩展到instance上，并绑定this
  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);
  // 将axios对象上的属性扩展到instance上
  // Copy context to instance
  utils.extend(instance, context);

  // 暴露出一个create方法，可以通过这个方法创建axios对象
  // Factory for creating new instances
  instance.create = function create(instanceConfig) {
    return createInstance(mergeConfig(defaultConfig, instanceConfig));
  };
  // 返回instance对象，这个就是新的axios对象了
  return instance;
}
// 新的axios对象
var axios = createInstance(defaults);
// Axios构造函数绑定到axios上
axios.Axios = Axios;
```
下面我们着重看一下`Axios.prototype.request`方法, 方法前面部分都是对配置的一些处理（就不细讲了），
然后从这里开始（也就是下面代码所示），这是`axios`的拦截器处理。

首先看一下请求拦截器：
请求拦截器会依次将`interceptor.fulfilled`和`interceptor.rejected`往数组（`requestInterceptorChain`）的前面放
```js
// lib\core\Axios.js
// 用于存储请求拦截器的回调
  var requestInterceptorChain = [];
  // 在对请求拦截器进行处理时，是采用同步处理方式还是采用异步处理的方式，默认在没有添加拦截器时时同步，添加拦截器后是异步
  var synchronousRequestInterceptors = true;
  // 遍历添加的拦截器
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    if (typeof interceptor.runWhen === 'function' && interceptor.runWhen(config) === false) {
      return;
    }
  // interceptor.synchronous来自添加拦截器时，所配置的选项
    synchronousRequestInterceptors = synchronousRequestInterceptors && interceptor.synchronous;
    // 这里将每个迭代器以fulfilled和rejected为一组的形式，依次放到数组的前面
    requestInterceptorChain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
```
响应拦截器处理:
响应拦截器会依次将`interceptor.fulfilled`和`interceptor.rejected`往数组（`responseInterceptorChain`）的后面放
```js
// 用于存储响应拦截器的回调
  var responseInterceptorChain = [];
  // 遍历添加的拦截器
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    // 这里将每个迭代器以fulfilled和rejected为一组的形式，依次放到数组的后面
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });
```

为什么会如此放置呢？在讨论这个问题之前，我们先来看一下这个拦截器（interceptors）是怎么存放我们注册的拦截器的。拦截器是在我们创建axios实例的时候创建的，它们都是同一个构造函数的两个不同的实例：
```js
// lib\core\Axios.js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    // 创建一个请求拦截器对象
    request: new InterceptorManager(),
    // 创建一个响应拦截器对象
    response: new InterceptorManager()
  };
}
```
拦截器的构造函数。当我们在调用axios.interceptors.request.use()时，其实就是向这个this.handlers的末尾注册一个拦截器回调
```js
//lib\core\InterceptorManager.js
function InterceptorManager() {
  // 存放添加的拦截器回调
  this.handlers = [];
}

// 注册拦截器
InterceptorManager.prototype.use = function use(fulfilled, rejected, options) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected,
    synchronous: options ? options.synchronous : false,
    runWhen: options ? options.runWhen : null
  });
  // 返回一个已注册拦截器的id，用于注销添加的拦截器
  return this.handlers.length - 1;
};

// 这是上面迭代拦截器的方法，拦截器对象自己实现了一个迭代方法
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```
接着我们来看下上面我们提出的问题，为什么会如此排列？
看代码上的注释
```js
  // 用来保存上一次promise.then处理后的结果
  var promise;
  // 将请求拦截器处理放到promise中进行异步处理
  if (!synchronousRequestInterceptors) {
    // 存储所有的操作，包括请求拦截器和响应拦截器的处理，dispatchRequest用来发起请求, 设置undefined是为了占位
    var chain = [dispatchRequest, undefined];
    // 请求拦截器和响应拦截器一前一后放入chain中
    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain = chain.concat(responseInterceptorChain);
```

这里做的处理，我觉得可以说是非常华丽了（一只菜汪路过）
1. 我们前面已经将所有的拦截器都放入了chain数组中，而且存放的格式是这样的
`[fulfilled1, rejected1, fulfilled2, rejected2, ...dispatchRequest, undefined, fulfilled1, rejected1,fulfilled2, rejected2, ...]`
2. 这里通过一个while循环来遍历这个数组（也就是chain），并进行处理
 
3. `promise = promise.then(chain.shift(), chain.shift());` 就是用来队列处理整个chain
4. 第一次循环`promise = promise.then(fulfilled1, rejected1)`处理第一个拦截器回调，等处理结束后赋值给promise，
5. 接着第二次进入循环`promise = promise.then(fulfilled2, rejected2)`
...
6. 当处理到`dispatchRequest, undefined`时，证明请求拦截器已经处理完成，开始发起请求，等请求结果回来，进入响应拦截器的处理
7. `promise = promise.then(fulfilled1, rejected1)`，同样的依次进行每个拦截器的回调

8. 最后返回这个经过请求拦截器处理，发起请求处理，和响应拦截器处理的结果 -- promise

> 注：在处理请求拦截器时，*最先添加到请求拦截器的回调，会最后执行；相反，对于响应拦截器的处理是，最先添加到响应拦截器的回调最先执行*

几句代码，但这分量确实足（学到了）
```js
    promise = Promise.resolve(config);
    while (chain.length) {
      promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
  }
```
> 这里是把请求拦截处理当做异步来处理的，对于同步处理，是将整个处理分为了两部分进行处理的，第一部分是请求拦截器部分，是同步处理的，第二部分是发起请求的部分和响应拦截器部分合在一起进行异步处理，就和上面的一样了

总结：可以看到在promise.then处理的时候，正好一个是fulfilled回调，一个是rejected回调，所以，我们将拦截器注册的回调也依次按这样排列就正好方便我们处理。不仅如此，我们将真正发起请求的处理也按着成对的逻辑并放置在中间，就能保证处理的顺序是按照：处理请求拦截器 => 发起请求 => 处理响应拦截器的顺序

