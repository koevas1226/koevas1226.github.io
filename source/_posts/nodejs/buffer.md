---
title: 浅析Node源码系列 - Buffer的内存分配
tags: [技术, 后端]
categories: NodeJS
date: 2020-11-10 19:43:02
---

## 前言
  `Buffer`作为 `NodeJS`中核心的一个模块用来进行处理网络二进制数据, 它是JavaScript的 [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) 类的子类, 也就是说, 你可以像使用 `Uint8Array` 一样去使用`Buffer`类:
  ```js
  const uint8 = new Uint8Array(8); // 分配1个字节
  uint8.from([1,2]);

  const alloc8 = Buffer.alloc(8); // 分配1个字节
  const buf = Buffer.from([3,4])
  ```
  此外, `NodeJS`还为`Buffer`拓展了很多功能。例如:

  ```js
  const buf = Buffer.from([0, 5]);

  console.log(buf.readInt16BE(0)); // 5
  console.log(buf.readInt16LE(0)); // 1280
  ```
  极大方便处理网络上的二进制数据。
## 内存分配
### Buffer.alloc(size[, fill[, encoding]])
  `Buffer.alloc()`允许手动申请一个内存块, 例如需要一个1KB的内存空间
  ```js
  const buf = Buffer.alloc(8 * 1024, 0); // 1kb填充0
  ```
### Buffer.allocUnsafe(size)
  它比`alloc`要快得多但是被标注为`unsafe`, 因为分配的内存可能含有脏数据。所以你可以这样使用
  ```js
  const buf = Buffer.allocUnsafe(8 * 1024);
  buf.fill(0); // clean data
  ```
### 区别和原理
  `Buffer.alloc()` 实际返回的是一个叫做 `FastBuffer`的实例:
  ```js
  Buffer.alloc = function alloc(size) {
    return new FastBuffer(size);
  };
  ```
  而这个`FastBuffer`实际上是内置的`Uint8Array`:

  ```js
  class FastBuffer extends Uint8Array {}
  ```
  到这里`alloc`的原理就清楚了, 那么`allocUnsafe`为什么会快? 
  ```js
  Buffer.allocUnsafe = function allocUnsafe(size) {
    return allocate(size);
  };
  function allocate(size) {
    // Buffer.poolSize = 8 * 1024  
    if (size < (Buffer.poolSize >>> 1)) {
      if (size > (poolSize - poolOffset))
        createPool();
      const b = new FastBuffer(allocPool, poolOffset, size);
      poolOffset += size;
      alignPool();
      return b;
    }
  }
  ```
  可以看到如果申请的内存小于`Buffer.poolsize >>> 1`的话，那么会调用`createPool()`去创建一个Buffer池, 这个内置的Buffer池默认大小是`8KB`, 也就是说申请小于`4kb`的内存时并且小于剩余偏移量就会从Buffer池返回一个`DataView`:

  ```js
  function createPool() {
    poolSize = Buffer.poolSize;
    allocPool = new FastBuffer(poolSize).buffer;  // 返回buffer的引用
    poolOffset = 0;
  }
  ```
  后置操作`alignPool()`会8位对齐内存, 偏移量永远保证整数字节,
  ```js
  function alignPool() {
    if (poolOffset & 0x7) {
      poolOffset |= 0x7;
      poolOffset++;
    }
  }
  ```
  如果需要的内存大于`8KB`那么就会直接申请而不再走这套逻辑。
## 举例
  ```js
  const buffer = Buffer.allocUnsafe(1024);
  // 等价于以下
  const bufferPool = new UInt8Array(8 * 1024);
  buffer = bufferPool.subArray(0,1024);
  ```
