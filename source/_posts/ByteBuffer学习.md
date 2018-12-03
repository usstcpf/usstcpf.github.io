---
title: ByteBuffer学习
date: 2018-10-17 21:23:57
tags: 
categories: 
- NIO学习笔记
---

NIO从零开始系列

<!-- more -->

# Buffer类

# ByteBuffer

- ByteBuffer是NIO里用得最多的Buffer，它包含两个实现方式：HeapByteBuffer是基于Java堆的实现，而DirectByteBuffer则使用了unsafe的API进行了堆外的实现

## 常见用法

## 重要属性

- byte[] buff: 内部用于缓存的数组
- mark: 为某一读过的位置做标记，便于某些时候回退到该位置。
- position: 当前读写的位置
- capacity: buffer的容量，分配好后大小不可变
- limit: 读写的上限，limit<=capacity。读模式下代表缓存中存在多少数据，写模式下代表最多能存入多少数据

**注意**

- `0 <= mark <= position <= limit <= capacity`
- 可以链式调用，比如 `ByteBuffer.allocate(10).put("123".getBytes());`

## 重要方法

- `ByteBuffer.allocate(int capacity)` 分配capacity大小的ByteBuffer

### ByteBuffer.allocate(int capacity)

```java
public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapByteBuffer(capacity, capacity);
}

HeapByteBuffer(int cap, int lim) {
    super(-1, 0, lim, cap, new byte[cap], 0);
}

ByteBuffer(int mark, int pos, int lim, int cap,
              byte[] hb, int offset)
{
    super(mark, pos, lim, cap);
    this.hb = hb;
    this.offset = offset;
}
```

allocate后，mark=-1,position=0,limit=capacity ![allocate后图](a.png)

### 