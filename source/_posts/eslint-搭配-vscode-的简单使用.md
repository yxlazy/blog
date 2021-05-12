---
title: eslint 搭配 vscode 的简单使用
date: 2021-05-12 12:22:13
tags: [javascript]
category: javascript
description: ESLint 是在 ECMAScript/JavaScript 代码中识别和报告模式匹配的工具，它的目标是保证代码的一致性和避免错误...
---
## 前言

刚开始时，由于嫌麻烦，并没有安装`eslint`，最近在新的项目上使用了`eslint`再配合`vscode`的插件，真是爽的不要太爽。因此打算写一篇简单的食用说明来记录食用过程

## 前期准备

没啥好准备的，作为开发肯定是具备`yarn`和`node`的，编辑器使用的是微软的亲儿子`vscode` 

然后新建一个文件夹`eslint-example`，`cd`进入这个文件夹并初始化一个`package.json`

初始化`package.json`命令

```bash
$ yarn init -y
```

文件结构

```
- eslint-example
	- package.json
```

## 配置

首先安装`eslint`，并初始化一个配置文件 

```bash
$ yarn add eslint --dev
$ ./node_modules/.bin/eslint --init
```

初始化完成后会在项目的根目录下生成一个配置文件`.eslintrc.js`（你的可能和我的不一样，但前缀都是一样的） 

> 关于配置文件的一些说明，配置文件可以使用`.js`，`.json`，`.yaml/.yml`后缀或者没有后缀的`.eslintrc`文件,也可以直接在`package.json`中添加一个`eslintConfig`属性进行配置。`eslint`会读取这个文件，优先级为从左到右依次查找文件格式，没有后缀的配置文件声明已废弃，不建议使用

```js
module.exports = {
	...
    //在这里添加自定义规则去覆盖默认规则
    "rules": {
        //要求或禁止使用分号代替 ASI
        "semi": ["error", "always"],
        //强制使用一致的缩进
        "indent": ["error", 2],
        //强制使用一致的反勾号、双引号或单引号
        "quotes": ["error", "double"],
        //禁用console
        "no-console": "warn" // 可以直接写错误类型
    }
    ...
}
```

`eslint`有三种错误规则，可以直接写规则类型也可以直接写数字，错误规则:

- `error`对应数字`2`
- `warn`对应数字`1`
- `off`对应数字`0` 

在`package.json`中添加`scripts`

```json
...
"script": {
    "lint": "eslint --ext .js src"
}
...
```

运行命令验证

## 配合vscode插件食用更舒适

在`vscode`的插件栏搜索`eslint`，安装`ESLint`插件

然后在`settings.json`中添加如下配置，对于更详细的配置请查看插件文档<sup>[2]</sup> 

```json
...
//为eslint开启自动修复，保存时将触发，但并不能自动修复所有的问题，但已经很爽了
"editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
},
```

## 参考

[1] [https://eslint.org/docs/user-guide/](https://eslint.org/docs/user-guide/) 

[2] [https://github.com/Microsoft/vscode-eslint#readme](https://github.com/Microsoft/vscode-eslint#readme) 