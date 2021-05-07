---
title: 'var, let, const详解'
date: 2021-03-21 11:44:18
tags: javascript
category: javascript
---
# var, let, const 声明变量详解

`var` 是`ES6`之前用来声明变量的关键字，而`let`和`const`是`ES6`之后的版本出现的。`let`和`const`是为了解决`var`带来的得怪异行为。

## var 声明

`var`声明的变量存在变量**提升（hoist**）

```js
function test() {
    console.log(a)
    var a = 10
}
test() //undefined
```

`var`声明的变量会先将变量`a`提升到作用域的顶部（在这里是`test`函数作用域)，并初始化为`undefined`，然后等到执行`var a = 10` 才真正进行变量的赋值，其实现过程相当于

```js
function test() {
    var a = undefined //变量提升到作用域顶部，并初始化为undefined
    console.log(a)
    a = 10
}
test() //undefined
```

如果在函数内部声明的变量没有加关键字，则声明的变量会被创建在全局作用域

```js
function test() {
    var a = 10;
    b = 'no var'
}
test() //undefined
console.log(a) //报错 ReferenceError: a is not defined
console.log(b) //正常输出 no var
```

反复多次使用`var`声明同一个变量也没有问题

```js
function foo() {
     var age = 16;
     var age = 26;
     var age = 36;
     console.log(age); 
}
foo(); // 36  
//引擎会将所有声明的变量提升，合并为一个声明，相当于
function foo() {
	var age = undefined
     age = 16;
     age = 26;
     age = 36;
     console.log(age); 
}
```

## let 和 const 声明

`let` 和 `const` 声明的范围是块作用域，而`var` 声明的范围是函数作用域。与`let`不同，`const`在声明时必须初始化。

```js
//var函数作用域
if (true) {
    var a = '函数作用域'
    console.log(a) //函数作用域
}
console.log(a) //函数作用域

//let和const块作用域
if (true) {
    let a = 'let 块作用域'
    const b = 'const 块作用域'
    console.log(a) //let 块作用域
    console.log(b) //const 块作用域
}
console.log(a) //报错 ReferenceError: a is not defined
console.log(b) //报错 ReferenceError: b is not defined
```

块作用域是函数作用域的子集，因此适用于`var`的作用域限制同样也适用于`let`

与`var`声明不同，`let`和`const`不允许同一个块作用域出现冗余声明

```js
var name;
var name;

let age;
let age; //报错 SyntaxError: Identifier 'age' has already been declared
```

但是对于在不同作用域中使用`let`和`const`声明是允许的

```js
let name = 'name';
if(true) {
    let name = 'another name'; //不会报错，且两者是不同的
}
```

在同一个作用域中，混合使用`let`和`var`声明同一个变量也是不允许的

let和const声明的变量不存在变量提升

```js
console.log(a); //报错 ReferenceError: a is not defined
console.log(b); //报错 ReferenceError: Cannot access 'b' before initialization

let a;
const b = 'const 声明的同时必须初始化';

//使用let 和 const 声明的变量，在执行到声明语句之前，会出现“暂时性死区”，在此阶段引用任何后面才声明的变量都会抛出 ReferenceError
```

在全局作用域下使用`let`声明的变量不会成为`window`对象的属性，但`var`声明的变量则会

```js
var name = 'Matt';
console.log(window.name); // 'Matt'
let age = 26;
console.log(window.age); // undefined 
```

const声明的变量只适用于它指向的变量的引用，即引用不可变。如果声明的是一个对象，则对象的属性是可以更改的，但引用不能更改

```js
const name = '我不可更改';
const  another = {
    name: '我是可以变的'
}

name = '我说了，我是不可变的'; //报错 TypeError: Assignment to constant variable.

another = '我声明时保存的是引用，引用也不能变的'; //报错 TypeError: Assignment to constant variable.
another.name = '我才是自由的';
console.log(another) 
//输出
//{
//    "name": "我才是自由的"
//}
```
