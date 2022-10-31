---
title: react-tiny-virtual-list源码解析
date: 2022-10-25 10:34:06
tags: [源码解析]
category: react
description: 对于长列表的渲染，通常的优化手段就是分页、懒加载和虚拟列表。而虚拟列表一直都是面试中的常见题，现在我们通过阅读react-tiny-virtual-list的源码来看看我们应该如何处理虚拟列表
keywords: 虚拟列表
---

## 前言
所谓虚拟列表，就是只渲染用户能实际看到的内容，也就是我们所说的视口，对于超过视口的部分就不渲染，从而避免渲染元素过多造成页面卡顿，大概效果如下

<img src='https://upload-images.jianshu.io/upload_images/20672535-fe03aee58f31b71d.png?imageMogr2/auto-orient/strip|imageView2/2/w/720/format/webp'>

> 图片来自这里https://www.jianshu.com/p/39404c94dbd0

## 源码解析
`react-tiny-virtual-list`支持垂直和水平方向上滚动，通过组件提供的`recomputeSizes()`方法，可支持加载动态高度的元素

### 部分参数分析

`width`和`height` 根据滚动的方向设置列表的宽度或者高度
`itemCount` 指定渲染列表的总数
`itemSize`支持纯数值，或者指定每一项元素高度的数据，或者一个返回高度的函数
`renderItem` 渲染元素，其参数包含了当前元素的`index`和`style`
`overscanCount` 要在可见项上方/下方渲染的额外缓冲区项数。调整此项可以帮助减少某些浏览器/设备上的滚动闪烁。
`estimatedItemSize` 估算的元素高度，用来支持动态高度

### 代码实现

首先我们看下组件的`render()`方法的实现

  ```typescript
  const STYLE_WRAPPER: React.CSSProperties = {
    overflow: 'auto',
    willChange: 'transform',
    WebkitOverflowScrolling: 'touch',
  };

  const STYLE_INNER: React.CSSProperties = {
    position: 'relative',
    width: '100%',
    minHeight: '100%',
  };
  // ...
  render() {
    const {
      estimatedItemSize,
      height,
      overscanCount = 3,
      renderItem,
      itemCount,
      itemSize,
      onItemsRendered,
      onScroll,
      scrollToIndex,
      scrollToAlignment,
      stickyIndices,
      style,
      width,
      ...props
    } = this.props;
    // 当前距离index为0的偏移，其计算值为，itemSize * 已经滚动的元素个数
    const {offset} = this.state;
    // 当前要渲染的元素start index 和stop index
    // sizeAndPositionManager 是 SizeAndPositionManager 的实例，用来计算渲染到页面的元素的工具
    const {start, stop} = this.sizeAndPositionManager.getVisibleRange({
      containerSize: this.props[sizeProp[scrollDirection]] || 0,
      offset,
      overscanCount,
    });
    const items: React.ReactNode[] = [];
    // 最外层容器的样式
    const wrapperStyle = {...STYLE_WRAPPER, ...style, height, width};
    // 每个元素的样式
    const innerStyle = {
      ...STYLE_INNER,
      [sizeProp[scrollDirection]]: this.sizeAndPositionManager.getTotalSize(),
    };

    // 设置元素的position为sticky
    if (stickyIndices != null && stickyIndices.length !== 0) {
      stickyIndices.forEach((index: number) =>
        items.push(
          renderItem({
            index,
            style: this.getStyle(index, true),
          }),
        ),
      );

      if (scrollDirection === DIRECTION.HORIZONTAL) {
        innerStyle.display = 'flex';
      }
    }
    // 要渲染的元素，传递index和style
    if (typeof start !== 'undefined' && typeof stop !== 'undefined') {
      for (let index = start; index <= stop; index++) {
        if (stickyIndices != null && stickyIndices.includes(index)) {
          continue;
        }

        items.push(
          renderItem({
            index,
            style: this.getStyle(index, false),
          }),
        );
      }
      // 这是个回调函数
      if (typeof onItemsRendered === 'function') {
        onItemsRendered({
          startIndex: start,
          stopIndex: stop,
        });
      }
    }

    return (
      <div ref={this.getRef} {...props} style={wrapperStyle}>
        <div style={innerStyle}>{items}</div>
      </div>
    );
  }
  ```

可以看到，这里最主要的操作是获取当前要渲染的元素，而这个操作是通过`this.sizeAndPositionManager.getVisibleRange()`函数计算出来的，而`this.sizeAndPositionManager`是在组件初始化时由`SizeAndPositionManager`创建的实例

```typescript
  // ...
  sizeAndPositionManager = new SizeAndPositionManager({
    // 列表总个数
    itemCount: this.props.itemCount,
    // 调用会返回列表的元素大小
    itemSizeGetter: this.itemSizeGetter(this.props.itemSize),
    // 列表元素的估算大小
    estimatedItemSize: this.getEstimatedItemSize(),
  });
  // ...
  itemSizeGetter = (itemSize: Props['itemSize']) => {
    // 可以看到获取元素大小，其实获取的是itemSize并不是estimatedItemSize
    return index => this.getSize(index, itemSize);
  };
```

`SizeAndPositionManager`是一个工具类，用于处理列表元素的信息,其构造函数，用于初始化一些变量，其中`this.itemSizeAndPositionData`用于缓存`item`的`size`和`offset`，`this.lastMeasuredIndex`用于保存最后一个计算元素时的`index`

```typescript
export default class SizeAndPositionManager {
  private itemSizeGetter: ItemSizeGetter;
  private itemCount: number;
  private estimatedItemSize: number;
  private lastMeasuredIndex: number;
  private itemSizeAndPositionData: SizeAndPositionData;

  constructor({itemCount, itemSizeGetter, estimatedItemSize}: Options) {
    this.itemSizeGetter = itemSizeGetter;
    this.itemCount = itemCount;
    this.estimatedItemSize = estimatedItemSize;

    // Cache of size and position data for items, mapped by item index.
    // 根据index，缓存item的大小和位置
    this.itemSizeAndPositionData = {};
    // Measurements for items up to this index can be trusted; items afterward should be estimated.
    this.lastMeasuredIndex = -1;
  }
  // ...
}
```

`getVisibleRange()`是用来获取当前渲染元素的`start index`和`stop index`

```typescript
  getVisibleRange({
    // 容器大小
    containerSize,
    // 当前偏移
    offset,
    // 额外多加载数据的个数
    overscanCount,
  }: {
    containerSize: number;
    offset: number;
    overscanCount: number;
  }): {start?: number; stop?: number} {
    // 计算元素总的高度
    const totalSize = this.getTotalSize();

    if (totalSize === 0) {
      return {};
    }
    // 最大偏移
    const maxOffset = offset + containerSize;
    // 查找与offset距离最近的元素的index，作为视口显示的第一个元素
    let start = this.findNearestItem(offset);

    if (typeof start === 'undefined') {
      throw Error(`Invalid offset ${offset} specified`);
    }
    // 获取当前start处 元素的信息
    const datum = this.getSizeAndPositionForIndex(start);
    offset = datum.offset + datum.size;

    let stop = start;
    // 计算视口显示的最后一个元素的index
    // 计算下一个元素的offset，等于当前元素的size + 当前元素的offset
    while (offset < maxOffset && stop < this.itemCount - 1) {
      stop++;
      offset += this.getSizeAndPositionForIndex(stop).size;
    }
    // 加载额外的数据，减少滚动闪烁
    if (overscanCount) {
      start = Math.max(0, start - overscanCount);
      stop = Math.min(stop + overscanCount, this.itemCount - 1);
    }

    return {
      start,
      stop,
    };
  }
```

组件在`componentDidMount`生命周期中，为容器组件挂载了滚动事件，用于处理滚动。当滚动触发时，去更新`offset`状态，从而触发新的计算后重新渲染元素

```typescript
  componentDidMount() {
    const {scrollOffset, scrollToIndex} = this.props;
    this.rootNode.addEventListener('scroll', this.handleScroll, {
      passive: true,
    });

    // ...
  }

  //...

  private handleScroll = (event: UIEvent) => {
    const {onScroll} = this.props;
    // 获取偏移
    const offset = this.getNodeOffset();
    if (
      offset < 0 ||
      this.state.offset === offset ||
      event.target !== this.rootNode
    ) {
      return;
    }
    // 更新offset，触发rerender
    this.setState({
      offset,
      scrollChangeReason: SCROLL_CHANGE_REASON.OBSERVED,
    });

    if (typeof onScroll === 'function') {
      onScroll(offset, event);
    }
  };
  // 容器元素的滚动距离(scrollTop/scrollLeft)，就是当前的offset
  private getNodeOffset() {
    const {scrollDirection = DIRECTION.VERTICAL} = this.props;

    return this.rootNode[scrollProp[scrollDirection]];
  }
```

## 总结
通过对源码的分析，可以看出，根据`start`和`stop`就可以计算出当前需要渲染的元素，要得到`start`值，就需要计算元素的偏移量`(offset)`，若要得到`stop`，需要将`start`之前的偏移量`(offset)`再加上容器的高度或宽度。在滚动时，需要监听容器的`scroll`事件，容器的`scrollTop/scrollLeft`就是当前要展示在容器中的元素的偏移量`(offset)`，然后再用前面的计算方式，就可以得到新的`start`和`stop`

## 参考

https://www.jianshu.com/p/39404c94dbd0

https://github.com/dwqs/blog/issues/71

https://github.com/clauderic/react-tiny-virtual-list