---
title: 使用Rollup,TypeScript,SCSS配置React组件库
date: 2023-01-03 17:10:44
tags: [typescript, rollup]
category: [typescript]
description: 想写一个React的组件库，但不知道该如何配置一个开发环境？这篇文章将告诉你答案
keywords: [rollup, typescript, react library]
---
> 本篇为翻译的文章，由于笔者能力有限，倘若阅读体验不好，可点击原文地址查看，原文地址：[Rollup Config for React Component Library With TypeScript + SCSS](https://www.codefeetime.com/post/rollup-config-for-react-component-library-with-typescript-scss/)


## 前言
在这篇文章中，我将尝试覆盖在构建`React`组件库时使用[`Rollup`](https://rollupjs.org/guide/en/)配置发挥作用的关键领域（特别是`TypeScript`和`SCSS`）。

同时，我将会为使用到的[`Rollup`插件](https://github.com/rollup/plugins)写一些说明，来说明插件的作用。

我绝对不是`Rollup`的大师，也不是构建`React`组件库的权威指南。我仅仅只是分享了用于构建我自己的`React`组件库（[Codefee-Kit](https://github.com/DriLLFreAK100/codefee-kit)）时的`Rollup`配置，这只是我自己的一个笔记，或许能帮到一些人。

顺便说一下，组件库托管在[Github](https://github.com/DriLLFreAK100/codefee-kit)上

## 动机
在此之前，我使用`webpack`去构建库。当时，我在获得第一个工作版本时遇到了很多麻烦。我仍然记得有个属性叫"library"，为了让`webpack`对库收集信息，你需要显示设置。我很傻，在我明白整个事情之前我在谷歌上搜索了很多信息。在整个踩坑过程中投入了大量时间。我当然不喜欢那里的配置文件的复杂性。嗯，是谁？哈哈哈

尽管如此，当时的工作版本并没有得到很好的优化。我的意思是，没有配置任何代码分割，构建的输出总是一个越来越大的`index.js`文件

最近，我终于又有了一些空闲时间！因此，我决定重新审视这个话题。在`Webpack`中对如何进行代码拆分进行了一些深挖，我只是觉得它太麻烦了。

最后，我决定彻底停止，然后跳到另一个流行的构建库的选择——`Rollup`！而且，它的配置简单得多，而且节省了我很多时间！天啊，我为什么不早点跳过去？！去我的！

有一句话是这样说的，"Rollup for libraries, Webpack for apps"。事实证明，在这一点上，它仍然非常重要！

> “Rollup for libraries, Webpack for apps” [https://medium.com/@PepsRyuu/why-i-use-rollup-and-not-webpack-e3ab163f4fd3](https://medium.com/@PepsRyuu/why-i-use-rollup-and-not-webpack-e3ab163f4fd3)

## 主题范围
请跳转到你感兴趣的部分

1. [Rollup 配置文件](#rollup-配置文件)
2. [Rollup 代码分割](#rollup-代码分割)
3. [Rollup 插件说明](#rollup-插件说明)

### Rollup 配置文件
这是适用于我的配置文件。那里有相当多的活动部件，但我尽可能地把它清理干净，这样就不会太伤眼睛了哈哈。。。😛

**rollup.config.js**
```javascript
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import typescript from '@rollup/plugin-typescript';

import postcss from "rollup-plugin-postcss";
import visualizer from 'rollup-plugin-visualizer';
import { terser } from 'rollup-plugin-terser';
import { getFiles } from './scripts/buildUtils';

const extensions = ['.js', '.ts', '.jsx', '.tsx'];

export default {
  input: [
    './src/index.ts',
    ...getFiles('./src/common', extensions),
    ...getFiles('./src/components', extensions),
    ...getFiles('./src/hooks', extensions),
    ...getFiles('./src/utils', extensions),
  ],
  output: {
    dir: 'dist',
    format: 'esm',
    preserveModules: true,
    preserveModulesRoot: 'src',
    sourcemap: true,
  },
  plugins: [
    resolve(),
    commonjs(),
    typescript({
      tsconfig: './tsconfig.build.json',
      declaration: true,
      declarationDir: 'dist',
    }),
    postcss(),
    terser(),
    visualizer({
      filename: 'bundle-analysis.html',
      open: true,
    }),
  ],
  external: ['react', 'react-dom'],
};
```
正如你所看到的，为了让`Rollup`工作，你仅仅只需要导出一个`JSON`对象。当然，如果你有多个配置需要配置，你也能导出一个数组。例如构建不同的目标文件时，像`umd`,`cjs`等

我在这里配置了4个属性：
- `input` - 打包文件的入口。可以是字符串或者字符串数组
- `output` - 打包文件的输出配置，例如配置打包输出的文件夹，配置`sourcemap`生成等
- `plugins` - 外部包调用，帮助操作、更改构建行为，例如`TypeScript, SCSS`等
- `external` - 不需要打包进构建包中的包，通常是指`peerDependencies`中设置的包，在我的例子中，，就是`react`和`react-dom`两个包

#### TypeScript 配置
以下是我的`tsconfig`文件。我有2个，一个是`Rollup`用于构建，另一个类似于基本配置，主要是用于开发

背后的原因是，我想排除我的[Storybook stories files](https://storybook.js.org/)被`TypeScript`转译。它只用于开发时，我需要`tsconfig`。因此，创建了一个带有`excludes`的单独的配置文件

**tsconfig.build.json**
```json
{
  "extends": "./tsconfig.json",
  "exclude": [
    "node_modules",
    "build",
    "dist",
    "scripts",
    "acceptance-tests",
    "webpack",
    "jest",
    "src/stories/**"
  ]
}
```

**tsconfig.json**
```json
{
  "compilerOptions": {
    "module": "esnext",
    "target": "es5",
    "lib": [ "es6", "dom" ],
    "sourceMap": true,
    "jsx": "react",
    "moduleResolution": "node",
    "rootDir": "src",
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "esModuleInterop": true,
    "baseUrl": "src/",
    "paths": {
      "common": ["src/common/*"],
      "components": ["src/components/*"],
      "hooks": ["src/hooks/*"],
      "utils": ["src/utils/*"]
    }
  },
  "exclude": [
    "node_modules",
    "build",
    "dist",
    "scripts",
    "acceptance-tests",
    "webpack",
    "jest"
  ],
  "types": ["typePatches"]
}
```

### Rollup 代码分割
在上面的配置中，我实际上使用了Rollup的[代码分割 (code-splitting) ](https://rollupjs.org/guide/en/#code-splitting)功能。

如果您注意到，我导出的`JSON`对象中的`input`属性实际上是字符串数组，而不是单个字符串。这实际上告诉`Rollup`将这些字符串中的每一个作为构建的单独入口点。所以，这就是代码分割构建。

数组中的所有这些字符串实际上是我在`src`文件夹中拥有的单个`.ts`和`.tsx`文件的相对路径。`getFiles`方法是我编写的一个实用方法，它帮助我深入检索所有扩展名为`.js、.jsx、.ts`和`.tsx`的文件。

下面是[`getFiles`](https://github.com/DriLLFreAK100/codefee-kit/blob/main/scripts/buildUtils.js)方法的代码:

```javascript
const fs = require('fs');

export const getFiles = (entry, extensions = [], excludeExtensions = []) => {
  let fileNames = [];
  const dirs = fs.readdirSync(entry);

  dirs.forEach((dir) => {
    const path = `${entry}/${dir}`;

    if (fs.lstatSync(path).isDirectory()) {
      fileNames = [
        ...fileNames,
        ...getFiles(path, extensions, excludeExtensions),
      ];

      return;
    }

    if (!excludeExtensions.some((exclude) => dir.endsWith(exclude))
      && extensions.some((ext) => dir.endsWith(ext))
    ) {
      fileNames.push(path);
    }
  });

  return fileNames;
};
```

### Rollup 插件说明
以下是我在上面的配置文件中使用的插件，以及使用它们后我的一些解释和理解。

[@rollup/plugin-node-resolve](https://www.npmjs.com/package/@rollup/plugin-node-resolve) 

对于`Rollup`，在`node_modules`中查找和捆绑第三方依赖项。这里所指的依赖关系是`package.json`文件中列出的依赖关系(`dependencies`)。

[@rollup/plugin-commonjs](https://www.npmjs.com/package/@rollup/plugin-commonjs) 

对于`Rollup`，将`CommonJS`模块转换为`ES6`，以便将其包含在`Rollup`捆绑包中。它通常与`@rollup/plugin-node-resolve` 一起使用，以捆绑`CommonJS`依赖项。

[@rollup/plugin-typescript](https://www.npmjs.com/package/@rollup/plugin-typescript) 

帮助以更简单的方式与`TypeScript`集成。包括将`TypeScript`转换为`JavaScript`等内容。

[rollup-plugin-postcss](https://www.npmjs.com/package/rollup-plugin-postcss)

与`PostCSS`集成。对于我的用例，它有助于将`CSS`和`SCSS`文件分别构建为`*.CSS.js`和`*.SCSS.js`文件。然后，当导入相应的组件时，这些文件会相应地注入`HTML head`标签（依赖于[style-inject](https://www.npmjs.com/package/style-inject)）
> 在我目前的开发中，我已经改用样式化组件，而倾向于在`JS`模式中尝试`CSS`。

[rollup-plugin-visualizer](https://www.npmjs.com/package/rollup-plugin-visualizer)

此插件用于帮助我们分析捆绑输出。它将为我们的检查生成一个很酷的可视化。在进行捆绑包大小优化时，它特别有用，因为它可以让我们可视化捆绑包文件的各个大小。

[rollup-plugin-terser](https://www.npmjs.com/package/rollup-plugin-terser)

它基本上是一个插件，用于集成[terser](https://www.npmjs.com/package/terser)的用法。它有助于缩小和压缩我们的输出`JavaScript`文件。

## 总结
在使用`Rollup`和`Webpack`构建使用`TypeScript`和`SCSS`的`React`组件库之后。。。我不得不说，只是为了这个目的而疯狂地使用`Rollup`。与`Webpack`相比，它更易于配置。

单是配置的复杂性就足以让我跳过去。我希望有一个配置文件，我可以在任何时间点轻松推理，而不是一些冗长复杂的东西，我可能会在几周内忘记正在做什么！

然而，我可能不一定对应用程序开发有同样的感觉，因为`Webpack`具有真正强大的热模块替换功能。这绝对是一个救命稻草，也是应用程序开发的必备工具。

构建工具的前景正在发生变化，如果第二天出现另一个事实上的`bundler`，并在社区中掀起风暴，这可能并不奇怪！但至少现在，`Rollup`是我的新情人😛

## 参考
1. [https://github.com/rollup/plugins](https://github.com/rollup/plugins)
2. [https://rollupjs.org/guide/en/](https://rollupjs.org/guide/en/)
3. [https://marcobotto.com/blog/compiling-and-bundling-typescript-libraries-with-webpack/](https://marcobotto.com/blog/compiling-and-bundling-typescript-libraries-with-webpack/)
4. [https://github.com/HarveyD/react-component-library](https://github.com/HarveyD/react-component-library)
5. [https://github.com/rollup/rollup-plugin-commonjs](https://github.com/rollup/rollup-plugin-commonjs)

