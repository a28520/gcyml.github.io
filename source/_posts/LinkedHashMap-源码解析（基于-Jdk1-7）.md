---
title: LinkedHashMap 源码解析（基于 JDK1.7）
date: 2019-01-22 21:23:30
tags:
---
## LinkedHashMap 简介

HashMap 是 Java 中非常常见的集合，但是 HashMap 有一个问题，就是**迭代HashMap的顺序并不是HashMap放置的顺序**。在一些应用场景中，我们是希望获得有序的 Map 的。
正是基于此，就有了 LinkedHashMap。LinkedHashMap 继承了 HashMap，在 HashMap 的基础上，通过内部维护一个双向链表，能够保证元素迭代的顺序。其顺序可以是插入顺序或者是访问顺序。

在这里对源码的分析，就不再重复 HashMap 中已存在的内容，只着重于 LinkedHashMap 中独有的部分。

## LinkedHashMap 中新增的属性

LinkedHashMap 中加入了两个属性：

1. **header：** 双向链表中的头
2. **accessOrder：** 访问顺序，默认为 false，即插入顺序。

## LinkedHashMap 内部键值对类 Entry

LinkedHashMap 的内部类 Entry 实现。
Entry 继承了 HashMap 的 Entry，加入了关于双向链表的属性和常用方法。

``` java
    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }

        // 从双向链表中删除
        private void remove() {
            before.after = after;
            after.before = before;
        }

        // 在指定节点前插入节点
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        // 若为访问顺序，则移动节点到链尾
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }

        // 删除
        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }
```

## LinkedHashMap 重写的方法

### 构造方法

构造方法调用了 HashMap 的方法，加入了初始化 `accessOrder` 为 false。以及重写了 `init` 方法。

init 方法是在 HashMap 构造方法中会调用到的方法。原来为空实现，这里加入了初始化双向链表的操作。

``` java
    @Override
    void init() {
        header = new Entry<>(-1, null, null, null);
        header.before = header.after = header;
    }
```

### 添加元素时对双向链表的维护

#### addEntry 方法

``` java
    void addEntry(int hash, K key, V value, int bucketIndex) {
        super.addEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

#### createEntry 方法

创建键值对放入桶中，在双向链表链尾中加入该节点。可以看到这里所有的插入操作，都是把元素插入到链表的头部，因此只要遍历 after，即为插入排序迭代。

``` java
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMap.Entry<K,V> old = table[bucketIndex];
        Entry<K,V> e = new Entry<>(hash, key, value, old);
        table[bucketIndex] = e;
        e.addBefore(header);
        size++;
    }
```

#### transfer 方法

transfer 方法只有 resize 方法会调用到，也就是 Map 动态扩充时。
这里就是遍历双向链表，用头插法把键值对插入到桶中。

``` java
    @Override
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            if (rehash)
                e.hash = (e.key == null) ? 0 : hash(e.key);
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }
```

### 访问顺序的实现原理

在调用 get 方法和添加元素时键值已存在时，均会调用 Entry 的 `recordAccess` 方法。可以看到，若为访问顺序时， `recordAccess` 会把该元素移动到链表的头部。

``` java
void recordAccess(HashMap<K,V> m) {
    LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
    if (lm.accessOrder) {
        lm.modCount++;
        remove();
        addBefore(lm.header);
    }
}
```

### 基于 LinkedHashMap 实现 FIFO 和 LRU

> **FIFO（First In First out）：** 先见先出，淘汰最先近来的页面，新进来的页面最迟被淘汰，完全符合队列。
**LRU（Least recently used）:** 最近最少使用，淘汰最近不使用的页面。

在前面的 `addEntry` 方法中，这里还调用了一个判断方法 `removeEldestEntry`，若为真，则删除链尾元素。**我们可以利用此方法来实现 `FIFO` 和 `LRU`**。
重写 `removeEldestEntry` 方法，若超过长度则删除链尾元素。
当为访问顺序时，由于长访问的都被移动到了链头位置，因此最少访问的元素在链表尾部。所以实现了 `LRU`。
为插入顺序时，最早插入的元素被移动到了链尾位置，删除链尾元素，则实现了 `FIFO`。

``` java
    void addEntry(int hash, K key, V value, int bucketIndex) {
        super.addEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

## 总结

LinkedHashMap 是在 HashMap 的基础上，加入了一个双向链表，所有的键值对元素都在这条双向链表上。通过维护双向链表，实现了顺序迭代。