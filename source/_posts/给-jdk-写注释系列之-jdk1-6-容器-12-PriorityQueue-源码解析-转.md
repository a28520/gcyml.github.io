---
title: 给 jdk 写注释系列之 jdk1.6 容器 (12)-PriorityQueue 源码解析 (转)
date: 2019-01-24 22:15:26
categories: "Java" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - PriorityQueue
---
> 本来想自己写的，网上查找资料，发现这篇 [给 jdk 写注释系列之 jdk1.6 容器(12)-PriorityQueue 源码解析](https://www.cnblogs.com/tstd/p/5125949.html) 文章写的很详细，思路清晰。自己再写估计也达不到原文的水平，于是直接转载原文了。

PriorityQueue 是一种什么样的容器呢？看过前面的几个 jdk 容器分析的话，看到 Queue 这个单词你一定会，哦~ 这是一种队列。是的，PriorityQueue 是一种队列，但是它又是一种什么样的队列呢？它具有着什么样的特点呢？它的底层实现方式又是怎么样的呢？我们一起来看一下。

PriorityQueue 其实是一个优先队列，什么是优先队列呢？这 **和我们前面讲的先进先出（First In First Out ）的队列的区别在于，优先队列每次出队的元素都是优先级最高的元素**。那么怎么确定哪一个元素的优先级最高呢，jdk 中使用堆这么一种数据结构，通过堆使得每次出队的元素总是队列里面最小的，而元素的大小比较方法可以由用户指定，这里就相当于指定优先级喽。

## 1. 二叉堆介绍

那么堆又是什么一种数据结构呢、它有什么样的特点呢？（以下见于百度百科）

1. 堆中某个节点的值总是不大于或不小于其父节点的值；
2. 堆总是一棵完全树。

常见的堆有二叉堆、斐波那契堆等。而 PriorityQueue 使用的便是二叉堆，这里我们主要来分析和学习二叉堆。

二叉堆是一种特殊的堆，二叉堆是完全二叉树或者是近似完全二叉树。二叉堆有两种：最大堆和最小堆。**最大堆：父结点的键值总是大于或等于任何一个子节点的键值；最小堆：父结点的键值总是小于或等于任何一个子节点的键值。**

说到二叉树我们就比较熟悉了，因为我们前面分析和学习过了二叉查找树和红黑树（TreeMap）。惯例，我们以最小堆为例，用图解来描述下什么是二叉堆。
![二叉堆](二叉堆.jpg)

上图就是一颗完全二叉树（二叉堆），我们可以看出什么特点吗，那就是在第 n 层深度被填满之前，不会开始填第 n+1 层深度，而且元素插入是从左往右填满。

基于这个特点，二叉堆又可以用数组来表示而不是用链表，我们来看一下：

![用数组表示二叉堆](用数组表示二叉堆.jpg)

通过 "用数组表示二叉堆" 这张图，我们可以看出什么规律吗？那就是，**基于数组实现的二叉堆，对于数组中任意位置的 n 上元素，其左孩子在 [2n+1] 位置上，右孩子 [2(n+1)] 位置，它的父亲则在 [(n-1)/2] 上，而根的位置则是[0]**

好了、在了解了二叉堆的基本概念后，我们来看下 jdk 中 PriorityQueue 是怎么实现的。

## 2. PriorityQueue 的底层实现

先来看下 PriorityQueue 的定义：

``` java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
```

我们看到 PriorityQueue 继承了 AbstractQueue 抽象类，并实现了 Serializable 接口，AbstractQueue 抽象类实现了 Queue 接口，对其中方法进行了一些通用的封装，具体就不多看了。

下面再看下 PriorityQueue 的底层存储相关定义：

``` java
    // 默认初始化大小
    privatestaticfinalintDEFAULT_INITIAL_CAPACITY = 11;

    // 用数组实现的二叉堆，下面的英文注释确认了我们前面的说法。
    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    private transient Object[] queue ;

    // 队列的元素数量
    private int size = 0;

    // 比较器
    private final Comparator<? super E> comparator;

    // 修改版本
    private transient int modCount = 0;
```

我们看到 jdk 中的 PriorityQueue 的也是基于数组来实现一个二叉堆，并且注释中解释了我们前面的说法。而 Comparator 这个比较器我们已经很熟悉了，我们说 PriorityQueue 是一个有限队列，他可以由用户指定优先级，就是靠这个比较器喽。

## 3. PriorityQueue 的构造方法

``` java
    /**
     * 默认构造方法，使用默认的初始大小来构造一个优先队列，比较器 comparator 为空，这里要求入队的元素必须实现 Comparator 接口
     */
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    /**
     * 使用指定的初始大小来构造一个优先队列，比较器 comparator 为空，这里要求入队的元素必须实现 Comparator 接口
     */
    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    /**
     * 使用指定的初始大小和比较器来构造一个优先队列
     */
    public PriorityQueue( int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        // 初始大小不允许小于 1
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        // 使用指定初始大小创建数组
        this.queue = new Object[initialCapacity];
        // 初始化比较器
        this.comparator = comparator;
    }

    /**
     * 构造一个指定 Collection 集合参数的优先队列
     */
    public PriorityQueue(Collection<? extends E> c) {
        // 从集合 c 中初始化数据到队列
        initFromCollection(c);
        // 如果集合 c 是包含比较器 Comparator 的 (SortedSet/PriorityQueue)，则使用集合 c 的比较器来初始化队列的 Comparator
        if (c instanceof SortedSet)
            comparator = (Comparator<? super E>)
                ((SortedSet<? extends E>)c).comparator();
        else if (c instanceof PriorityQueue)
            comparator = (Comparator<? super E>)
                ((PriorityQueue<? extends E>)c).comparator();
        //  如果集合 c 没有包含比较器，则默认比较器 Comparator 为空
        else {
            comparator = null;
            // 调用 heapify 方法重新将数据调整为一个二叉堆
            heapify();
        }
    }

    /**
     * 构造一个指定 PriorityQueue 参数的优先队列
     */
    public PriorityQueue(PriorityQueue<? extends E> c) {
        comparator = (Comparator<? super E>)c.comparator();
        initFromCollection(c);
    }

    /**
     * 构造一个指定 SortedSet 参数的优先队列
     */
    public PriorityQueue(SortedSet<? extends E> c) {
        comparator = (Comparator<? super E>)c.comparator();
        initFromCollection(c);
    }

    /**
     * 从集合中初始化数据到队列
     */
    private void initFromCollection(Collection<? extends E> c) {
        // 将集合 Collection 转换为数组 a
        Object[] a = c.toArray();
        // If c.toArray incorrectly doesn't return Object[], copy it.
        // 如果转换后的数组 a 类型不是 Object 数组，则转换为 Object 数组
        if (a.getClass() != Object[].class)
            a = Arrays. copyOf(a, a.length, Object[]. class);
        // 将数组 a 赋值给队列的底层数组 queue
        queue = a;
        // 将队列的元素个数设置为数组 a 的长度
        size = a.length ;
    }
```

构造方法还是比较容易理解的，第四个构造方法中，如果填入的集合 c 没有包含比较器 Comparator，则在调用 initFromCollection 初始化数据后，在调用 heapify 方法对数组进行调整，使得它符合二叉堆的规范或者特点，具体 heapify 是怎么构造二叉堆的，我们后面再看。

那么怎么样调整才能使一些杂乱无章的数据变成一个符合二叉堆的规范的数据呢？

## 4. 二叉堆的添加原理及 PriorityQueue 的入队实现

我们回忆一下，我们在说红黑树 TreeMap 的时候说，红黑树为了维护其红黑平衡，主要有三个动作：左旋、右旋、着色。那么二叉堆为了维护他的特点又需要进行什么样的操作呢。

我们再来看下二叉堆（最小堆为例）的特点：

1. **父结点的键值总是小于或等于任何一个子节点的键值。**
2. **基于数组实现的二叉堆，对于数组中任意位置的 n 上元素，其左孩子在 [2n+1] 位置上，右孩子 [2(n+1)] 位置，它的父亲则在 [n-1/2] 上，而根的位置则是[0]。**

为了维护这个特点，二叉堆在添加元素的时候，需要一个 "上移" 的动作，什么是 "上移" 呢，我们继续用图来说明。

![二叉堆入队示意1](二叉堆入队示意1.jpg)

![二叉堆入队示意2](二叉堆入队示意2.jpg)

结合上面的图解，我们来说明一下二叉堆的添加元素过程：

1. 将元素 2 添加在最后一个位置（队尾）（图 2）。
2. 由于 2 比其父亲 6 要小，所以将元素 2 上移，交换 2 和 6 的位置（图 3）；
3. 然后由于 2 比 5 小，继续将 2 上移，交换 2 和 5 的位置（图 4），此时 2 大于其父亲（根节点）1，结束。

注：这里的节点颜色是为了凸显，应便于理解，跟红黑树的中的颜色无关，不要弄混。。。

看完了这 4 张图，是不是觉得二叉堆的添加还是挺容易的，那么下面我们具体看下 PriorityQueue 的代码是怎么实现入队操作的吧。

``` java
    /**
     * 添加一个元素
     */
    public boolean add(E e) {
        return offer(e);
    }

    /**
     * 入队
     */
    public boolean offer(E e) {
        // 如果元素 e 为空，则排除空指针异常
        if (e == null)
            throw new NullPointerException();
        // 修改版本 + 1
        modCount++;
        // 记录当前队列中元素的个数
        int i = size ;
        // 如果当前元素个数大于等于队列底层数组的长度，则进行扩容
        if (i>= queue .length)
            grow(i + 1);
        // 元素个数 + 1
        size = i + 1;
        // 如果队列中没有元素，则将元素 e 直接添加至根（数组小标 0 的位置）
        if (i == 0)
            queue[0] = e;
        // 否则调用 siftUp 方法，将元素添加到尾部，进行上移判断
        else
            siftUp(i, e);
        return true;
    }
```

jdk 中，不是直接将根元素删除，然后再将下面的元素做上移，重新补充根元素；而是找出队尾的元素，并在队尾的位置上删除，然后通过根元素的下移，给队尾元素找到一个合适的位置，最终覆盖掉跟元素，从而达到删除根元素的目的。这样做在一些情况下，会比直接删除在上移根元素，或者直接下移根元素再调整队尾元素的位置少操作一些步奏（比如上面图解中的例子，不信你可以试一下 ^_^）。

明白了二叉堆的入队和出队操作后，其他的方法就都比较简单了，下面我们再来看一个二叉堆中比较重要的过程，二叉堆的构造。

## 6. 堆的构造过程

我们在上面提到过的，堆的构造是通过一个 heapify 方法，下面我们来看下 heapify 方法的实现。

``` java
    /**
     * Establishes the heap invariant (described above) in the entire tree,
     * assuming nothing about the order of the elements prior to the call.
     */
    private void heapify() {
        for (int i = (size>>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```

这个方法很简单，就这几行代码，但是理解起来却不是那么容器的，我们来分析下。

假设有一个无序的数组，要求我们将这个数组建成一个二叉堆，你会怎么做呢？最简单的办法当然是将数组的数据一个个取出来，调用入队方法。但是这样做，每次入队都有可能会伴随着元素的移动，这么做是十分低效的。那么有没有更加高效的方法呢，我们来看下。

为了方便，我们将上面我们图解中的数组去掉几个元素，只留下 7、6、5、12、10、3、1、11、15、4（顺序已经随机打乱）。ok、那么接下来，我们就按照当前的顺序建立一个二叉堆，暂时不用管它是否符合标准。

`int a = [7, 6, 5, 12, 10, 3, 1, 11, 15, 4];`

![初始化数组](初始化数组.jpg)

我们观察下用数组 a 建成的二叉堆，很明显，对于叶子节点 4、15、11、1、3 来说，它们已经是一个合法的堆。所以只要最后一个节点的父节点，也就是最后一个非叶子节点 a[4]=10 开始调整，然后依次调整 a[3]=12，a[2]=5，a[1]=6，a[0]=7，分别对这几个节点做一次 "下移" 操作就可以完成了堆的构造。ok，我们还是用图解来分析下这个过程。

![构建二叉堆1](构建二叉堆1.jpg)

![构建二叉堆2](构建二叉堆2.jpg)

![构建二叉堆3](构建二叉堆3.jpg)

我们参照图解分别来解释下这几个步奏：

1. 对于节点 a[4]=10 的调整（图 1），只需要交换元素 10 和其子节点 4 的位置（图 2）。
2. 对于节点 a[3]=12 的调整，只需要交换元素 12 和其最小子节点 11 的位置（图 3）。
3. 对于节点 a[2]=5 的调整，只需要交换元素 5 和其最小子节点 1 的位置（图 4）。
4. 对于节点 a[1]=6 的调整，只需要交换元素 6 和其最小子节点 4 的位置（图 5）。
5. 对于节点 a[0]=7 的调整，只需要交换元素 7 和其最小子节点 1 的位置，然后交换 7 和其最小自己点 3 的位置（图 6）。

至此，调整完毕，建堆完成。

再来回顾一下，PriorityQueue 的建堆代码，看看是否可以看得懂了。

``` java
    private void heapify() {
        for (int i = (size>>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```

`int i = (size>>> 1) - 1`，这行代码是为了找寻最后一个非叶子节点，然后倒序进行 "下移"siftDown 操作，是不是很显然了。

到这里 PriorityQueue 的基本操作就分析完了，明白了其底层二叉堆的概念及其入队、出队、建堆等操作，其他的一些方法代码就很简单了，这里就不一一分析了。

PriorityQueue 完！

参见：
[给 jdk 写注释系列之 jdk1.6 容器 (11)-Queue 之 ArrayDeque 源码解析](http://www.cnblogs.com/tstd/p/5112177.html)

[给 jdk 写注释系列之 jdk1.6 容器 (9)-Strategy 设计模式之 Comparable&Comparator 接口](http://www.cnblogs.com/tstd/p/5090401.html)

参考资料：
[图解数据结构（8）——二叉堆
java 实现选择排序 (直接选择排序、堆排序)](http://www.cnblogs.com/yc_sunniwell/archive/2010/06/28/1766751.html)