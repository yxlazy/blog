---
title: FormData多文件上传并同时添加其他数据
date: 2021-09-10 11:14:05
tags: [javascript]
category: javascript
description: 如何使用FormData实现文件上传呢？单文件上传和多文件上传呢？
---

## 前言
在做项目时，有一个接口要求上传文件的同时能携带其它数据，在这里使用的是`FormData`对象来进行操作。简单介绍一下这个对象。
`FormData` 接口提供了一种表示表单数据的键值对 `key/value` 的构造方式，并且可以轻松的将数据通过`XMLHttpRequest.send()` 方法发送出去，本接口和此方法都相当简单直接。如果送出时的编码类型被设为 `"multipart/form-data"`，它会使用和表单一样的格式。(以上出自[这里](https://developer.mozilla.org/zh-CN/docs/Web/API/FormData)) 
## 单文件上传
这里使用`FormData`的`append()`方法，该方法会向`FormData`对象中添加新的属性值
```html
  <input id='file' type='file'/>
  <button id='upload'>上传</button>
  <script>
    const input = document.getElementById('file');
    const uploadBtn = document.getElementById('upload');
    let files = null;

    input.onchange = (e) => {
      const file = input.files[0];
      files = file;
    }

    const uploadFile = (file) => {
      const formData = new FormData();
      formData.append('file', file);
      fetch('https://xxx.com', {
        method: 'POST', 
        body: formData
      })
      .then(res => {
        //在这里处理请求成功
      })
      .catch(err => {
        //在这里处理请求失败
      })
    }
    uploadBtn.onclick = () => {
      if (!files) {
        alert('请选择文件')
        return
      }
      uploadFile(files);
    }
  </script>
```
### 多文件上传
如果`FormData`的属性值中已经存在对应的属性值也不会被覆盖，而是会将这个新值添加到已有集合的后面
```html
+ <input id='file' type='file' multiple/>
  <button id='upload'>上传</button>
  <script>
    const input = document.getElementById('file');
    const uploadBtn = document.getElementById('upload');
    let files = null;

    input.onchange = (e) => {
-     const file = input.files[0];
-     files = file;
+     const files = input.files
+     files = files;
    }

-   const uploadFile = (file) => {
+   const uploadFile = (files) => {
      const formData = new FormData();
-     formData.append('file', file);
//多文件上传，formData的append方法，对于同一个key会将新添加的key/value添加到后面，不会覆盖已经存在的key
+     for (const file of files) {
+       formData.append('files', file);
+     } 
      fetch('https://xxx.com', {
        method: 'POST', 
        body: formData
      })
      .then(res => {
        //在这里处理请求成功
      })
      .catch(err => {
        //在这里处理请求失败
      })
    }
    uploadBtn.onclick = () => {
      if (!files) {
        alert('请选择文件')
        return
      }
      uploadFile(files);
    }
  </script>
```

### 多文件上传并携带其他数据
```html
  <input id='file' type='file' multiple/>
  <button id='upload'>上传</button>
  <script>
    const input = document.getElementById('file');
    const uploadBtn = document.getElementById('upload');
    let files = null;

    input.onchange = (e) => {
      const files = input.files
      files = files;
    }

-   const uploadFile = (files) => {
+   const uploadFile = (files, payload) => {
      const formData = new FormData();
      for (const file of files) {
        formData.append('files', file);
      } 
      //对于其他附加的数据，分别增加到FormData中即可
+     for (const key in payload) {
+       formData.append([key], payload[key]);
+     }
      fetch('https://xxx.com', {
        method: 'POST', 
        body: formData
      })
      .then(res => {
        //在这里处理请求成功
      })
      .catch(err => {
        //在这里处理请求失败
      })
    }
    uploadBtn.onclick = () => {
      if (!files) {
        alert('请选择文件')
        return
      }
-     uploadFile(files);
+     uploadFile(files, {id: 'upload'});
    }
  </script>
```
> `FormData`对象的`set()`方法和`append()`方法比较，`set()`方法指定的键如果存在，会使用新值覆盖原来的值，而`append()`方法会把新值添加到已有值集合的后面

## 参考
[1] https://developer.mozilla.org/zh-CN/docs/Web/API/FormData