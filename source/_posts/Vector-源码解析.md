---
title: Vector 源码解析
date: 2019-01-26 23:46:29
categories: "Java" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Vector
---
## 关于 synchronized

在 Java 中 synchronized 可用来给对象和方法或者代码块加锁，当它锁定一个方法或者一个代码块的时候，同一时刻最多只有一个线程执行这段代码。

而 synchronized 底层是通过使用对象的监视器锁（monitor）来确保同一时刻只有一个线程执行被修饰的方法或者代码块。可以用锁和钥匙来解释，被 synchronized 修饰的方法或者代码块是一把锁，这把锁是归对象所有的，当一个线程需要执行这些方法或者代码块的时候，锁就被钥匙插上了，所以其他线程就不能执行这些方法或者代码块。（实际情况还要更加复杂，这里只是便于理解，实际上 synchronized 的锁是可重入的）

## Vector 同步
它的实现与 ArrayList 类似，但是使用了 synchronized 进行同步。

``` java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
```
``` java
public synchronized E get(int index) {
    if (index>= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index);
}
```
## 与 ArrayList 的比较
- Vector 是同步的，因此开销就比 ArrayList 要大，访问速度更慢。最好使用 ArrayList 而不是 Vector，因为同步操作完全可以由程序员自己来控制；
- Vector 每次扩容请求其大小的 2 倍空间，而 ArrayList 是 1.5 倍。

## 感谢

本文所有内容均节选自下面两篇文章，感谢分享。

[Java 容器](https://github.com/CyC2018/CS-Notes/blob/master/docs/notes/Java%20%E5%AE%B9%E5%99%A8.md#vector)

[Java 中的 synchronized](https://blog.csdn.net/a158123/article/details/78607964)