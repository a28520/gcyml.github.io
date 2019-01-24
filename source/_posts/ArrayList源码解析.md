---
title: ArrayList源码解析
date: 2018-06-11 12:05:18
categories: "Java" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - ArrayList
---
## ArrayList简介

ArrayList是一个实现了List接口的动态数组，其容量能够动态增长，其允许包括null在内的所有元素。它继承了AbstractList，实现了List、RandomAccess, Cloneable, java.io.Serializable。
因为其基于数组实现的特性，所以随机访问 get 和 set 效率较高，但是在于添加和删除操作，由于要移动数据，因此效率会低些。
需要注意的是，ArrayList中的操作不是线程安全的！如果多线程同时访问 ArrayList ，而其中有个线程对表结构做了修改，则它可能会抛出错误。所以，建议在单线程中才使用ArrayList，而在多线程中可以选择Vector或者CopyOnWriteArrayList。下面我们就了解一下ArrayList一些常用的方法属性，以及一些需要注意的地方。

> 本文转自[ArrayList源码分析（基于JDK8）](https://blog.csdn.net/fighterandknight/article/details/61240861)，本人仅在自己的理解上做了些许的修改。  

## ArrayList属性

属性中比较重要的两个就是用于记录当前数组长度的 size，以及元素的缓存数组 elementData，除此之外还有一个经常用到的属性就是从AbstractList继承过来的modCount属性，代表ArrayList集合的修改次数。

``` java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable {  
    // 序列化id  
    private static final long serialVersionUID = 8683452581122892189L;  
    // 默认初始的容量  
    private static final int DEFAULT_CAPACITY = 10;  
    // 用于空实例的共享空数组实例
    private static final Object[] EMPTY_ELEMENTDATA = {};
    // 为默认大小的空实例构建的共享空数组实例。
    // 和 EMPTY_ELEMENTDATA 区别在于添加第一个元素需要膨胀多少
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // 当前数据对象存放地方，当前对象不参与序列化  
    transient Object[] elementData;  
    // 当前数组长度  
    private int size;  
    // 数组最大长度  
    private static final int MAX_ARRAY_SIZE = 2147483639;  

    // 省略方法。。  
}
```
## ArrayList 构造方法

### 无参构造方法

如果不传入参数，则使用默认无参构建方法创建ArrayList对象，如下：

``` java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

*注意：此时我们创建的ArrayList对象中的elementData中的长度是1，size是0,当进行第一次add的时候，elementData将会变成默认的长度：10.*

### 带int类型的构造函数

传入的参数用于指定初始数组长度，若参数大于等于0，则给elementData 赋予对应长度的Object数组，
若参数小于0，则抛出IllegalArgumentException异常，构造方法如下：

``` java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

### 带Collection对象的构造函数

构造一个包含指定 Collection 的元素的列表，这些元素是按照该 collection 的迭代器返回它们的顺序排列的。
其内部实现大致为：

1. 将 Collection 对象转为数组，然后将数组的地址的赋给elementData。
2. 更新size的值，同时判断size的大小，如果是size等于0，直接将空对象EMPTY_ELEMENTDATA的地址赋给elementData
3. 如果size的值大于0，且 elementData 的 class 不等于 Object 数组的 class，则执行Arrays.copyOf方法，把 Collection对象的内容（可以理解为深拷贝）copy到 elementData 中。

*注意：this.elementData = arg0.toArray(); 这里执行的简单赋值时浅拷贝，所以要执行Arrays,copy 做深拷贝*

``` java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

## 常用方法

### add 方法

#### add(E e) 方法

此方法将指定的元素添加到列表的尾部中，内部实现逻辑如下：

1. 确保数组已使用长度（size）加1之后足够存入下一个数据
2. 修改次数modCount 标识自增1，如果当前数组已使用长度（size）加1后的大于当前的数组长度，则调用grow方法，增长数组，grow方法会将当前数组的长度变为原来容量的1.5倍。
3. 确保新增的数据有地方存储之后，则将新元素添加到位于size的位置上，size自增1
4. 成功返回 true

添加元素方法入口：

``` java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

确保添加的元素有地方存储：

``` java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
    }

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

将修改次数（modCount）自增1，判断是否需要扩充数组长度，判断条件就是用当前所需的数组最小长度与数组的长度对比，如果大于0，则增长数组长度。

``` java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

如果当前的数组已使用空间（size）加1之后大于数组长度，则增大数组容量，扩大为原来的1.5倍。

``` java
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

#### add(int index, E element) 方法

这个方法其实和上面的add类似，该方法可以按照元素的位置，指定位置插入元素，其后的元素往后移动一位，具体的执行逻辑如下：

1. 越界检查，否则抛出异常
2. 确保数组已使用长度（size）加1之后足够存入下一个数据
3. 修改次数（modCount）标识自增1，如果当前数组已使用长度（size）加1后的大于当前的数组长度，则调用grow方法，增长数组
4. grow方法会将当前数组的长度变为原来容量的1.5倍。
5. 确保有足够的容量之后，使用System.arraycopy 将需要插入的位置（index）后面的元素统统往后移动一位。
6. 将新的数据内容存放到数组的指定位置（index）上

``` java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
    elementData[index] = element;
    size++;
}
```

*注意：使用该方法的话将导致指定位置后面的数组元素全部重新移动，即往后移动一位。*

### get方法

返回此列表中指定位置上的元素

``` java
public E get(int index) {
    rangeCheck(index);  // 若 index >= size，则抛出异常

    return elementData(index);
}
```

### set方法

``` java
public E set(int index, E element) {
    rangeCheck(index); 若 index >= size，则抛出异常

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
  }
```

### contains方法

直接调用了 indexOf 方法，若返回小于 0 则返回 false。

``` java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

``` java
/**
 * 返回此列表中首次出现的指定元素的索引, 或如果此列表不包含元素，则返回 -1。
 */
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

### remove方法

#### 根据索引remove

1. 越界检查，否则抛出异常
2. modCount 自增1
3. 将指定位置（index）上的元素保存到oldValue
4. 将指定位置（index）上的元素都往前移动一位
5. 将最后面的一个元素置空，好让垃圾回收器回收
6. 将原来的值oldValue返回

``` java
/**
 * 移除此列表中指定位置上的元素。向左移动所有后续元素（将其索引减 1）。
 */
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

*注意：调用这个方法不会缩减数组的长度，只是将最后一个数组元素置空而已。*

#### 根据对象remove

循环遍历所有对象，得到对象所在索引位置，然后调用fastRemove方法，执行remove操作

``` java
/**
* 移除此列表中首次出现的指定元素（如果存在）。如果列表不包含此元素，则列表不做改动。
*/
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

定位到需要remove的元素索引，先将index后面的元素往前面移动一位（调用System.arraycooy实现），然后将最后一个元素置空。

``` java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

### clear方法

modCount 自增，当前数组长度内的所有元素赋为 null，并把 size 置 0

``` java
/**
 * 移除此列表中的所有元素。此调用返回后，列表将为空。
 */
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

### sublist方法

我们看到代码中是创建了一个ArrayList 类里面的一个内部类SubList对象，传入的值中第一个参数时this参数，其实可以理解为返回当前list的部分视图，真实指向的存放数据内容的地方还是同一个地方，如果修改了sublist返回的内容的话，那么原来的list也会变动。

``` java
public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, 0, fromIndex, toIndex);
}
```

### trimToSize方法

1. 修改次数加1
2. 将elementData中空余的空间（包括null值）去除，例如：数组长度为10，其中只有前三个元素有值，其他为空，那么调用该方法之后，数组的长度变为3。Arrays.copyOf返回的是一个新数组，因此达到了节省空间的效果

``` java
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```

### iterator方法

interator方法返回的是一个内部类，由于内部类的创建默认含有外部的this指针，所以这个内部类可以调用到外部类的属性。

``` java
public Iterator<E> iterator() {
    return new Itr();
}
```

#### Iterator 的 next方法

一般的话，调用完iterator之后，我们会使用iterator做遍历，这里使用next做遍历的时候有个需要注意的地方，就是调用next的时候，可能会引发ConcurrentModificationException，当修改次数，与期望的修改次数（调用iterator方法时候的修改次数）不一致的时候，会发生该异常，详细我们看一下代码实现：

``` java
@SuppressWarnings("unchecked")
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

expectedModCount这个值是在用户调用ArrayList的iterator方法时候确定的，但是在这之后用户add，或者remove了ArrayList的元素，那么modCount就会改变，那么这个值就会不相等，将会引发ConcurrentModificationException异常，这个是在多线程使用情况下，比较常见的一个异常。

``` java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

#### Iterator 的 remove方法

cursor 记录了当前元素索引，lastRet则记录了上一元素的索引，调用迭代器自带的remove方法，先调用了ArrayList的remove方法后，
会把操作次数modCount赋给expectedModCount，因此不会引发异常。

``` java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

## System.arraycopy 方法

| 参数    | 说明                   |
|:--------|:-----------------------|
| src     | 原数组                 |
| srcPos  | 原数组                 |
| dest    | 目标数组               |
| destPos | 目标数组的起始位置     |
| length  | 要复制的数组元素的数目 |

## Arrays.copyOf方法

original - 要复制的数组
newLength - 要返回的副本的长度
newType - 要返回的副本的类型

其实Arrays.copyOf方法返回的是一个全新数组，其底层也是调用System.arraycopy实现的，源码如下：

``` java
public static int[] copyOf(int[] original, int newLength) {   
   int[] copy = new int[newLength];   
   System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));   
   return copy;   
}
```

## 小结

ArrayList总体来说比较简单，不过ArrayList还有以下一些特点：

* ArrayList自己实现了序列化和反序列化的方法，因为它自己实现了 private void writeObject(java.io.ObjectOutputStream s)和 private void readObject(java.io.ObjectInputStream s) 方法
* ArrayList基于数组方式实现，无容量的限制（会扩容）添加元素时可能要扩容（所以最好预判一下），删除元素时不会减少容量（若希望减少容量，trimToSize()），**删除元素时，将删除掉的位置元素置为null，下次gc就会回收这些元素所占的内存空间** 。
* 线程不安全
* add(int index, E element)：添加元素到数组中指定位置的时候，需要将该位置及其后边所有的元素都整块向后复制一位
* get(int index)：获取指定位置上的元素时，可以通过索引直接获取（O(1)）
* remove(Object o)需要遍历数组
* remove(int index)不需要遍历数组，只需判断index是否符合条件即可，效率比remove(Object o)高
* contains(E)需要遍历数组
* 使用iterator遍历可能会引发多线程异常