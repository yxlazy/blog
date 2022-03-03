---
title: 使用husky+lint-staged规范代码格式
date: 2022-03-03 10:21:58
tags: [代码规范]
category: other
---

1. 安装`eslint, prettier`和`lint-staged`
>此处并不打算列出eslintrc.js和prettierrc.js的具体配置，具体配置请自行百度

```bash
yarn add eslint prettier lint-staged --dev
npx husky-init && yarn
```

2. 更改`pre-commit.sh`下的命令为
`npx lint-staged`

> `pre-commit` 是执行`git commit`时的`hook`

3. 在package.json下配置

```js
{
  "lint-staged": {
    "**/*.{js, jsx, ts, tsx}": [
      <!-- 每次commit时，都将更改的文件进行格式检查并格式化 -->
      "eslint --fix",
      <!-- 如果有格式化的文件，就将其添加到暂存区，方便commit提交文件 -->
      "git add"
    ]
  }
}
```

4. 在eslint使用prettier

```bash
yarn add eslint-plugin-prettier eslint-config-prettier --dev
```

5. 在eslintrc.js文件添加如下配置

```js
{
  <!-- 一定要放到最后面，才能覆盖其他配置 注：v8之后就是使用如下格式-->
  extends: ['prettier'],
  plugins: ['prettier'],
  rules: {
    <!-- 使用prettier, 默认会使用.prettierrc.js中的规则 -->
    "prettier/prettier": "error"
  }
}
```
