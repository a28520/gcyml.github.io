---
title: HashMap 源码解析（基于jdk 1.7）
date: 2019-01-17 21:44:24
categories: "Java" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - HashMap
---
## HashMap 简介

HashMap 是基于哈希表实现的映射表。并允许使用 null 值和 null 键。（除了不同步和允许使用 null 之外，HashMap 类与 Hashtable 大致相同。）此类不保证映射的顺序，特别是它不保证该顺序恒久不变。

值得注意的是HashMap不是线程安全的，如果想要线程安全的HashMap，可以通过Collections类的静态方法synchronizedMap获得线程安全的HashMap。

``` java
Map map = Collections.synchronizedMap(new HashMap());
```

## HashMap 属性

HashMap 中有两个比较重要的属性：**容量**和**加载因子**。这两个变量能影响到 HashMap 的性能。**容量是哈希表中桶的数量**，初始容量只是哈希表在创建时的容量，默认初始容量等于 16。**加载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度**。当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 rehash 操作（即重建内部数据结构）。

可以注意到源码中要求容量必须为 2 次幂。由于 HashMap 是基于哈希实现的，为了存取高效，要尽量较少碰撞，就是要尽量把数据分配均匀，每个链表长度大致相同，这个实现就在把数据存到哪个链表中的算法；这里是用了取模来实现 `hash % length`。但是直接求余效率不如位移运算，源码中做了优化，改成了 `hash & (length-1)`。而这种做法下，要保证取出来的值分布尽量均匀，`length-1` 二进制上的 1 必须尽可能的多，可知 length 是 2 次幂时成立。

例如长度为 9 时候，3&(9-1)=0  2&(9-1)=0 ，都在0上，碰撞了；
例如长度为 8 时候，3&(8-1)=3  2&(8-1)=2 ，不同位置上，不碰撞；

``` java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
    // 初始默认容量 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

    // 最大容量值
    static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    // 映射表未扩充前的空映射表
    static final Entry<?,?>[] EMPTY_TABLE = {};

    // 映射表数组。其长度必须为 2 次幂
    transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

    // 映射表大小
    transient int size;

    // 扩容表的阈值（容量 * 加载因子）。如果table == EMPTY_TABLE，那么阈值等于创建表的初始容量。
    int threshold;

    // 加载因子
    final float loadFactor;

    // 记录数组发生结构修改的次数
    transient int modCount;

    // 阈值的默认值
    static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;

    ......
}
```

## HashMap 构造方法

### HashMap(int initialCapacity, float loadFactor)

这个构造方法指定了容量和加载因子，内部实现基本就是参数检查和赋值。

``` java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        this.loadFactor = loadFactor;
        threshold = initialCapacity;
        init();
    }

```

### HashMap(int initialCapacity)

调用了第一种的构造方法，传入指定的容量和默认加载因子。

### HashMap()

调用了第一种的构造方法，传入默认容量 16 和默认加载因子 0.75。

### HashMap(Map<? extends K, ? extends V> m)

构造一个映射关系与指定 Map 相同的新 HashMap。
这里先调用 `HashMap(int initialCapacity, float loadFactor)` 方法指定合适的容量，
而后再调用 `inflateTable` 方法把容量变为 2 次幂，并创建桶数组实例，
最后遍历参数中的 map，在新 HashMap 中写入元素。

``` java
    public HashMap(Map<? extends K, ? extends V> m) {
        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                      DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
        inflateTable(threshold);

        putAllForCreate(m);
    }
```

``` java
    /**
     * Inflates the table.
     */
    private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        int capacity = roundUpToPowerOf2(toSize);

        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }
```

取出目标映射表的视图，迭代调用 `putForCreate`。

``` java
    private void putAllForCreate(Map<? extends K, ? extends V> m) {
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
            putForCreate(e.getKey(), e.getValue());
    }
```

取得桶下标 `h & (length-1)`，若已存在，则更新，不存在则调用 `createEntry` 创建键值对到对应桶中。

**注：** 由于哈希碰撞总是无法完全避免的，因此为了解决这一问题，HashMap 用到了 `拉链法`。`拉链法` 的实现比较简单，将链表和数组相结合。数组中的元素称为桶，桶中装的是链表。若遇到哈希冲突，则将冲突的值加到链表中即可。

``` java
    private void putForCreate(K key, V value) {
        int hash = null == key ? 0 : hash(key);
        int i = indexFor(hash, table.length);

        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                e.value = value;
                return;
            }
        }

        createEntry(hash, key, value, i);
    }
```

创建键值对到桶中。

``` java
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```

**注：** 在 `createEntry` 方法中，可以注意到创建新的键值对时，**新的键值对会插入到桶中链表的头部，而非尾部**。之所以选择插入到头部，是为了避免尾部遍历。理论上认为新插入的元素权重更高。

## 常用方法

### get 方法

由于 HashMap 键值均支持 null，由于 null 不支持 equals 方法，因此需要指定的方法来获取 键 为 null 的键值对。

``` java
    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }
```

在 HashMap 中用下标为 0 的桶存放键为 null 的键值对。

``` java
    private V getForNullKey() {
        if (size == 0) {
            return null;
        }
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null)
                return e.value;
        }
        return null;
    }
```

取出对应的桶下标，遍历桶中链表得到键值对。

``` java
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```

### put 方法

在此映射中关联指定值与指定键。如果该映射以前包含了一个该键的映射关系，则旧值被替换。内部实现流程如下：

1. 若为空表，则调用 `inflateTable` 扩容数组。
2. 若键为 null，则调用 putForNullKey 来进行 put 操作。
3. 取得桶下标，若已存在键值对，则更新值。
4. 桶中不存在该键的映射，则创建新的键值对实例到桶中。由于此时映射表结构发生改变，modCount自增。

``` java
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

针对键为 nul 的 put 方法。除了是指定从下标为 0 的桶中操作以外，和前面没有区别。

``` java
    private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);
        return null;
    }
```

添加键值对。首先判断当前长度是否超过了阈值，若超过则动态扩充到现有数组长度的 2 倍。然后调用 `createEntry` 创建键值对。`createEntry` 已经在 `HashMap(Map<? extends K, ? extends V> m)` 讲过，这里不再赘述。

``` java
    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }
```

动态扩充方法。首先当前容量若超过容量最大值，则把触发扩充的阈值置为 int 最大值。
创建一个新的指定容量 newCapacity 大小的数组，键值对拷贝到新数组中，更新阈值。

``` java
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
```

拷贝键值对到新数组中。

``` java
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

### putAll 方法

该方法用来追加另一个 Map 对象到当前 Map 集合对象，它会把另一个 Map 集合对象中的所有内容添加到当前 Map 集合对象。

``` java
    public void putAll(Map<? extends K, ? extends V> m) {
        int numKeysToBeAdded = m.size();
        if (numKeysToBeAdded == 0)
            return;

        if (table == EMPTY_TABLE) {
            inflateTable((int) Math.max(numKeysToBeAdded * loadFactor, threshold));
        }

        if (numKeysToBeAdded > threshold) {
            int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
            if (targetCapacity > MAXIMUM_CAPACITY)
                targetCapacity = MAXIMUM_CAPACITY;
            int newCapacity = table.length;
            while (newCapacity < targetCapacity)
                newCapacity <<= 1;
            if (newCapacity > table.length)
                resize(newCapacity);
        }

        for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
            put(e.getKey(), e.getValue());
    }
```

### remove方法

``` java
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }
```

取得桶下标，遍历链表找到键值对，从链表中删除该键值对。modCount 自增，size 自减。

``` java
    final Entry<K,V> removeEntryForKey(Object key) {
        if (size == 0) {
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }

```

### containsKey

判断映射表中是否存在该键。调用 `getEntry` 取出对应的桶下标，遍历桶中链表得到键值对（代码在 `get 方法` 中已贴出）。

``` java
    public boolean containsKey(Object key) {
        return getEntry(key) != null;
    }
```

### containsValue

遍历所有键值对，若值存在于映射表中，则返回 true。

``` java
    public boolean containsValue(Object value) {
        if (value == null)
            return containsNullValue();

        Entry[] tab = table;
        for (int i = 0; i < tab.length ; i++)
            for (Entry e = tab[i] ; e != null ; e = e.next)
                if (value.equals(e.value))
                    return true;
        return false;
    }

    /**
     * Special-case code for containsValue with null argument
     */
    private boolean containsNullValue() {
        Entry[] tab = table;
        for (int i = 0; i < tab.length ; i++)
            for (Entry e = tab[i] ; e != null ; e = e.next)
                if (e.value == null)
                    return true;
        return false;
    }

```

### clone 方法

调用父类的 clone 方法，扩充数组，初始化变量，浅拷贝键值对。

``` java
    public Object clone() {
        HashMap<K,V> result = null;
        try {
            result = (HashMap<K,V>)super.clone();
        } catch (CloneNotSupportedException e) {
            // assert false;
        }
        if (result.table != EMPTY_TABLE) {
            result.inflateTable(Math.min(
                (int) Math.min(
                    size * Math.min(1 / loadFactor, 4.0f),
                    // we have limits...
                    HashMap.MAXIMUM_CAPACITY),
               table.length));
        }
        result.entrySet = null;
        result.modCount = 0;
        result.size = 0;
        result.init();
        result.putAllForCreate(this);

        return result;
    }
```

### clear 方法

modCount 自增，数组中不再引用链表，指向 null。（不再引用使得 JVM 垃圾回收发生作用），置 size 为 0。

``` java
    public void clear() {
        modCount++;
        Arrays.fill(table, null);
        size = 0;
    }
```

## HashMap 视图相关

`Collection<V> values()` 和 `Set<K> keySet()` 两个视图方法的迭代方法都是通过继承内部抽象类`HashIterator` 实现。

### 内部抽象类 HashIterator

``` java
    private abstract class HashIterator<E> implements Iterator<E> {
        Entry<K,V> next;        // next entry to return
        int expectedModCount;   // For fast-fail
        int index;              // current slot
        Entry<K,V> current;     // current entry

        HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }

        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }
    }
```

`HashIterator` 创建实例时，取得第一个键值对和桶下标。

迭代方法 `nextEntry` 遍历链表和桶得到下一个键值对。

迭代器安全的删除键值对的方法 `remove`，保持了 modCount 一致。

## 小结

HashMap 由于是基于哈希表实现的，因此为了避免哈希碰撞冲突，让数据分布均匀，保证映射表的容量必须为 2 次幂。

另外，由于哈希碰撞总是无法完全避免，采用了 数组 + 链表的形式，即 “拉链法”， 让哈希碰撞的键值对组成链表。同时为了避免尾部遍历，新插入的键值对是插入到链表头部的。

![HashMap 实现原理](HashMap实现原理.png)

容量和加载因子是 HashMap 中很重要的两个参数，会影响到 HashMap 的性能。当数组长度超过了 `容量 * 加载因子` 时，HashMap 会触发自动扩充，从而哈希表将具有大约两倍的桶数。