---
title: 定型数组(typed array)
date: 2021-04-09 14:12:33
tags: javascript
category: javascript
---
# 定型数组（typed array)
定型数组`（typed array）`是`ECMAScript`新增的结构，目的是提升向原生库传输数据的效率。实际上，`JavaScript`并没有`"TypedArray"`类型，它所指的其实是一种特殊的包含数值类型的数组

## ArrayBuffer

`ArrayBuffer`是所有定型数组及视图引用的**基本单位**。

> `SharedArrayBuffer`是`ArrayBuffer`的一个变体，可以无须复制就在执行上下文间传递它

```js
const buf = new ArrayBuffer(2) //在内存中分配2个字节
console.log(buf)	//ArrayBuffer(2) {}
console.log(buf.byteLength) //2 //返回分配的字节长度
```

`ArrayBuffer`一经创建就不能再调整大小

```js
const buf = new ArrayBuffer(16)
const buf2 = buf.slice(4, 12) //但可以使用slice()复制部分到新的实例中
console.log(buf2.byteLength) //8
```

`ArrayBuffer`分配的堆内存可以被当成垃圾回收，不用手动释放（相比于`C/C++`的`malloc()`）

## DataView

`DataView`用于读写`ArrayBuffer`的视图。专为文件`I/O`和网络`I/O`设计，其`API`支持对缓冲数据的高度控制，但较其他类型的视图，性能差一些。

必须对已有的`ArrayBuffer`进行读/写才能创建`DataView`实例

```js
const buf = new ArrayBuffer(16)
const dv = new DataView(buf) //默认使用整个ArrayBuffer
console.log(dv) //DataView(16) {} 
console.log(dv.byteLength) //16
console.log(dv.byteOffset) //0
console.log(dv.buffer === buf) //true

//使用部分ArrayBuffer
const dv2 = new DataView(buf, /*offset*/2, /*length*/4)
console.log(dv2) //DataView(4) {}
console.log(dv2.byteLength) //4
console.log(dv2.byteOffset) //2
console.log(dv2.buffer === buf) //true
```

读取缓冲，需要几个`ElementType`组件，类似于`C`中的类型

```js
const buf = new ArrayBuffer(16)
const dv = new DataView(buf, 0, 8)

//说明DataView的实例默认值初始值为0
console.log(dv.getInt8(0)) //0 //以8位有符号整数进行读取,读取第一个字节
console.log(dv.getInt8(1)) //0
console.log(dv.getInt16(0)) //0 //以16位有符号整数进行读取,读取两个字节
//写入
dv.setInt8(1, 255) //给第二个字节写入255，二进制每一位都是1
console.log(dv.getInt8(1)) //-1 //由于是有符号的读，会被当做二补数（即反码）处理，字节开头的1被当作符号位
console.log(dv.getUint8(1)) //255 //无符号读取
```

除了上述的类型（`Int8, Uint8, Int16`）还有`Uint16,Int32, Uint32, Float32, Float64 `。默认的字节序为大端字节序。传递第二个参数可以决定字节序，为`true`时，则启用小端字节序

> `DataView` 只支持两种约定：大端字节序和小端字节序。大端字节序也称为“网络字节序”，意思是最高有效位保存在第一个字节，而最低有效位保存在最后一个字节。小端字节序正好相反，即最低有效位保存在第一个字节，最高有效位保存在最后一个字节。

`DataView`的读、写操作都必须满足充足的缓冲区，如果越界会抛出`RangError`

```js
const buf = new ArrayBuffer(2)
const dv = new DataView(buf) //这里为2个字节
//尝试越界读取4个字节
dv.getInt32(0) //错误 RangeError: Offset is outside the bounds of the DataView
```

## 定型数组

定型数组是另一种形式的`ArrayBffer`视图。设计定型数组的目的就是提高与 `WebGL` 等原生库交换二进制数据的效率。

```js
//创建一个12字节的缓冲
const buf = new ArrayBuffer(12)
//创建一个引用buf缓冲的Int32Array
const ints = new Int32Array(buf)
console.log(ints) //Int32Array(3) [0, 0, 0]
//length属性，打印长度
console.log(ints.length) //3
//与DataView一样，也有一个指向关联缓冲的引用
console.log(ints.buffer) //ArrayBuffer(12) {}

//创建一个指定长度的Int32Array
const ints1 = new Int32Array(6)
console.log(ints1.length) //6

//传入数组初始化Int32Array
const ints2 = new Int32Array([1, 2, 3, 4])
console.log(ints2.length) //4
console.log(ints2[1]) //2

//通过<ElementType>.from和<ElementType>.to初始化Int32Array
const ints3 = Int32Array.from(ints2)
const ints4 = Int32Array.of(10, 20, 30, 40)
console.log(ints3.length) //4
console.log(ints4.length) //4
console.log(ints3[1]) //20
console.log(ints4[1]) //20

//BYTES_PER_ELEMENT属性返回类型数组中每个元素的大小
console.log(Int32Array.BYTES_PER_ELEMENT) //4
console.log(Float64Array.BYTES_PER_ELEMENT) //8
console.log(ints4.BYTES_PER_ELEMENT) //4
```
定型数组支持数组的方法和属性，包括`Symbol.iterator`符号属性，但是没有`concat(), poop(), push(), shift(), unshift(), splice()`。定型数组提供了两个方法可以实现复制数据：`set()`和`subarray()`

`set()`从提供的数组或定型数组中把值复制到当前的定型数组中指定索引位置

```js
const container = new Int8Array(8)
//复制到从偏移量为2的位置开始的地方，默认偏移量为0
container.set(new Int8Array([2, 3, 4, 5]), 2)
console.log(container) //[0, 0, 2, 3, 4, 5, 0, 0]

//溢出会抛出错误
container.set(new Int8Array([3, 4,5]), 7) //错误 RangeError: offset is out of bounds
```
`subarray()`会基于原始定型数组返回复制的值组成新的定型数组
```js
const source = new Int8Array([1, 2, 3, 4, 5, 6, 7, 8])
//复制索引2到4，左闭右开
const newSource = source.subarray(2, 4)
console.log(newSource) // [3, 4]
console.log(newSource) //Int8Array(2) [3, 4]
```
除了`8`种元素类型，还有一种”夹板“数组类型：`Uint8ClampedArray`，用于解决溢出问题。该类型不允许任何方向溢出，超出最大值`255`的值会被向下舍入为`255`，而小于最小值`0`的值会被向上舍入为`0`
```js
const clampedInts = new Uint8ClampedArray([-1, 0, 255, 256])
console.log(clampedInts) //[0, 0, 255, 255]
```
> 除非做的和`canvas`相关开发，否则不要使用它