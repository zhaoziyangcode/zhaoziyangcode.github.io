---
layout:     post                    # 使用的布局（不需要改）
title:      ArrayList的实现原理               # 标题 
subtitle:   ArrayList的底层原理 #副标题
date:       2018-12-29              # 时间
author:     BY                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 数据结构
    - Java
---


### ArrayList
##### ArrayList实现于List，RandomAccess接口，可以插入空数据，也支持随机访问
ArrayList相当于动态数据，最重要的参数分别是：elementData数组，以及size大小在其调用add方法的时候：

```
public boolean add(E e){
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
- 先进行扩容校验ensureCapacityInternal(size + 1)
- 将插入的值放在elementData的尾部并将size+1
```
private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }
```
- 先判断ArrayList默认的元素存储是否为空，若为空的话则取默认大小和想要初始化大小两个数中的最大值然后再调用 ensureExplicitCapacity(minCapacity)

```
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
- 如果说初始化长度>elementData数组的长度,将会触发ArrayList的扩容机制然后调用grow()方法

```
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
- 当ArrayList扩容的时候，首先会设置新的存储能力为原来的1.5倍
- 如果扩容之后还是不能满足要求则MAX_ARRAY_SIZE比较，求取最大值， 
如果MAX_ARRAY_SIZE大小的能力还是不能满足则通过hugeCapacity()方法获取ArrayList能允许的最大值

```
private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
- 从hugeCapacity方法看出，ArrayList最大的存储能力：存储元素的个数为整型的范围。 
确定ArrayList扩容之后最新的可存储元素个数时，调用 
elementData = Arrays.copyOf(elementData, newCapacity); 
实现elementData数组的扩容，整个流程就是ArrayList的自动扩容机制工作流程
##### 如果是调用add(index index,E e)向指定位置添加元素的话

```
public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //复制，向后移动
        System.arraycopy(elementData, index, elementData, index + 1,
        size - index);
        elementData[index] = element;
        size++;
    }
```
- 可以看到第一步依然是进行扩容校验
- 接着对数组进行复制，将element插入index位置上，并将index位置后面的元素依次向后移动一个位置
#### 可以发现，ArrayList扩容的时候要做大量的操作，其中还包括一些调用native方法，就不可避免的需要做一些IO操作，所以在初始化ArrayList的时候最好给一个初始值的大小，这样可以避免后面增加元素的时候做频繁的扩容。
