---
title: 并发 - JUC - PriorityBlockingQueue - 源码剖析
date: 2016-08-28 00:06:25
categories:
    - Concurrent
    - JUC
tags:
    - Netease
    - Concurrent
    - JUC
    - AQS
---

{% note info %}
本文将通过剖析`PriorityBlockingQueue`的源码来介绍其实现原理
关于`ReentrantLock`的基本内容请参考「并发 - JUC - ReentrantLock - 源码剖析」，本文不再赘述
{% endnote %}

<!-- more -->

# 基础

## 概述
`PriorityBlockingQueue`是`支持优先级`的`无界阻塞队列`
`PriorityBlockingQueue`默认采用`自然升序`，可以在`初始化`时通过传入`Comparator`指定排序规则
`PriorityBlockingQueue`底层通过**`二叉堆`**实现优先级队列

## 二叉堆

### 结构
结构类似于二叉树，父节点的键值`总是`小于等于（或大于等于）子节点的键值，父节点的`左子树`和`右子树`都是一个`二叉堆`
最大堆：父节点的键值总是大于等于子节点的键值
最小堆：父节点的键值总是小于等于子节点的键值
![priorityblockingqueue_min_max_heap_1.png](http://otr5jjzeu.bkt.clouddn.com/priorityblockingqueue_min_max_heap_1.png)

### 存储
二叉堆一般采用`数组`存储，`a[n]`的`左子节点`为`a[2*n+1]`，`a[n]`的`右子节点`为`a[2*n+2]`，`a[n]`的`父节点`为`a[(n-1)/2]`
![priorityblockingqueue_heap_array_1.png](http://otr5jjzeu.bkt.clouddn.com/priorityblockingqueue_heap_array_1.png)

# 源码分析

## 核心结构
```Java
public class PriorityBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
    // 数组默认大小
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    // 数组最大大小
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    // 表示堆的数组
    private transient Object[] queue;
    // 对大小
    private transient int size;
    // 排序规则
    private transient Comparator<? super E> comparator;
    // 用于所有公共操作的锁
    private final ReentrantLock lock;
    // 非空等待条件
    private final Condition notEmpty;
    // 扩容时采用的自旋锁
    private transient volatile int allocationSpinLock;
    // 用于序列化，主要为了兼容之前的版本，本文不关注该特性
    private PriorityQueue<E> q;
}
```

## 构造函数
```Java
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

public PriorityBlockingQueue(int initialCapacity) {
   this(initialCapacity, null);
}

public PriorityBlockingQueue(int initialCapacity, Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}

public PriorityBlockingQueue(Collection<? extends E> c)
```

## add
```Java
public boolean add(E e) {
    return offer(e);
}
```

### offer
```Java
public boolean offer(E e) {
    if (e == null)
        // 不接受null元素
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock(); // 获取独占锁
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        // 扩容
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            // 节点上冒，采用自然排序
            siftUpComparable(n, e, array);
        else
            // 节点上冒，采用cmp指定的排序规则
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        // 唤醒等待非空条件的线程
        notEmpty.signal();
    } finally {
        lock.unlock(); // 释放独占锁
    }
    return true;
}
```

### tryGrow
```Java
// 扩容
// 先释放独占锁，允许多线程以CAS的方式创建新数组，然后重新竞争锁，进行数组复制
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); // 先释放独占锁
    Object[] newArray = null;
    // CAS方式抢占自旋锁
    if (allocationSpinLock == 0 && UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset, 0, 1)) {
        // 当前线程持有自旋锁，并发时只有一个线程能进入到这里
        try {
        // 新容量
        int newCap = oldCap + ((oldCap < 64) ? (oldCap + 2) : (oldCap >> 1));
        if (newCap - MAX_ARRAY_SIZE > 0) { // 可能溢出
            int minCap = oldCap + 1;
            if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                // 等价于：Integer.MAX_VALUE - 7 < oldCap <=Integer.MAX_VALUE
                throw new OutOfMemoryError();
            newCap = MAX_ARRAY_SIZE; // 采用最大容量
        }
        if (newCap > oldCap && queue == array)
            // 1. 既然扩容，必然要求newCap > oldCap
            // 2. finally会释放自旋锁，其他线程就有可能获得自旋锁，
            //    而刚刚释放自旋锁的线程可能已经扩容成功了，因此需要判断queue==array
            newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0; // 释放自旋锁
        }
    }
    if (newArray == null)
        // newArray==null说明其他线程正在进行扩容处理，让出CPU资源
        Thread.yield();
    lock.lock(); // 获取独占锁
    if (newArray != null && queue == array) {
        // newArray!=null说明新数组内存分配已经完成（但不一定是当前线程完成的）
        // queue==array说明其他线程尚未完成扩容
        // 反过来说：
        // 1. 如果newArray==null，说明其他线程正在进行扩容处理，进入下一个循环判断，
        //    由于当前线程持有独占锁，其他线程无法完成queue=newArray语句，只能等待，
        //    因此当前线程会再次进入tryGrow（释放独占锁）
        // 2. 如果queue!=array，说明其他线程已经完成了扩容，无需再次扩容
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap); // 数组复制
    }
}
```

### siftUpComparable
```Java
// 节点上冒，采用自然排序
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) { // k=0表示array[0]为根节点，无法继续上冒，直接退出
        int parent = (k - 1) >>> 1; // 父节点：(n-1)/2
        Object e = array[parent];
        if (key.compareTo((T) e) >= 0)
            // 子节点 >= 父节点，退出循环，最小堆
            break;
        // 子节点 < 父节点，将原先父节点的值移动到子节点的位置
        // 这时array[parent]形成可覆盖的空穴，下一次循环时（可能）被覆盖
        array[k] = e; 
        k = parent; // 准备下一次上冒
    }
    array[k] = key;
}
```

### siftUpUsingComparator
```Java
// 节点上冒，采用cmp指定的排序规则
// 跟siftUpComparable非常类似，不再赘述
private static <T> void siftUpUsingComparator(int k, T x, Object[] array, Comparator<? super T> cmp) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (cmp.compare(x, (T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = x;
}
```

### 逻辑示意图
```Java
public static void main(String[] args) {
    int initCap = 15;
    PriorityBlockingQueue<Integer> queue = new PriorityBlockingQueue(initCap);
    IntStream.range(0, initCap).forEach(i -> queue.add(2 * i + 1));
    queue.add(2);
}
```
![priorityblockingqueue_heap_add_1.png](http://otr5jjzeu.bkt.clouddn.com/priorityblockingqueue_heap_add_1.png)


## poll
```Java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 获取独占锁
    try {
        return dequeue();
    } finally {
        lock.unlock(); // 释放独占锁
    }
}
```

### dequeue
```Java
private E dequeue() {
    int n = size - 1;
    if (n < 0)
        // 队列为空，返回null
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0]; // 暂存第一个节点，用于返回，因为会在下冒过程中被覆盖
        E x = (E) array[n]; // 暂存最后一个节点
        array[n] = null; // 置空最后一个节点
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            // 节点下冒，采用自然排序
            siftDownComparable(0, x, array, n);
        else
            // 节点下冒，采用cmp指定的排序规则
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}
```

### siftDownComparable
```Java
// 节点下冒，采用自然排序
// k：需要填充的位置，poll操作时，默认为0
// x：需要插入的元素，poll操作时，为原尾节点
// array：表示堆的数组
// n：现在堆大小，为原先堆大小-1，尾节点保存在x
private static <T> void siftDownComparable(int k, T x, Object[] array, int n) {
    if (n > 0) {// n==0，说明原先堆只有一个节点，无需下冒
        Comparable<? super T> key = (Comparable<? super T>)x;
        int half = n >>> 1;
        // k>=half表示array[k]为叶子节点，无法继续下冒，直接退出
        // 在dequeue中，n=(size-1)，尾节点为a[size-1]=a[n]，dequeue会置空a[n]
        // 1. 假若尾节点为其父节点的左子节点，即a[n]=a[2*j+1]，父节点为a[j]，half=n>>>1=j，
        //    置空a[n]后，a[j]失去了唯一的子节点，成为叶子节点，因为a[k<half=j]为非叶子节点
        // 2. 假若尾节点为其父节点的右子节点，即a[n]=a[2*j+2]，父节点为a[j]，half=n>>>1=j+1，
        //    置空a[n]后，a[j]仍拥有左子节点，a[j]为非叶子节点，a[j+1]为叶子节点，因此a[k<half=j+1]为非叶子节点
        while (k < half) { 
            // child表示a[k]左右子节点中较小节点的索引，暂时表示左子节点的索引
            int child = (k << 1) + 1;
            // c表示a[k]左右子节点中较小节点的值，暂时表示左子节点的值
            Object c = array[child];
            // a[k]右子节点的索引
            int right = child + 1;
            if (right < n && ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                // a[k]右子节点存在+a[k]左子节点的值大于a[k]右子节点的值，更新child和c
                c = array[child = right];
            if (key.compareTo((T) c) <= 0)
                // 如果key（即原尾节点的值）不大于a[k]左右子节点中较小节点，就没必要继续下冒，退出循环
                break;
            array[k] = c;
            k = child;
        }
        array[k] = key; // 将原尾节点重新加入堆
    }
}
```

### siftDownUsingComparator
```Java
// 节点下冒，采用cmp指定的排序规则
// // 跟siftDownComparable非常类似，不再赘述
private static <T> void siftDownUsingComparator(int k, T x, Object[] array, int n, Comparator<? super T> cmp) {
    if (n > 0) {
        int half = n >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = array[child];
            int right = child + 1;
            if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                c = array[child = right];
            if (cmp.compare(x, (T) c) <= 0)
                break;
            array[k] = c;
            k = child;
        }
        array[k] = x;
    }
}
```

### 逻辑示意图
```Java
public static void main(String[] args) {
    PriorityBlockingQueue<Integer> queue = new PriorityBlockingQueue(15);
    Arrays.asList(1,             // 第1层
         2, 3,                   // 第2层
         7, 8, 4, 5,             // 第3层
         10, 11, 12, 13, 6, 9)   // 第4层
         .forEach(i -> queue.add(i));
    queue.poll();
}
```
![priorityblockingqueue_heap_poll.png](http://otr5jjzeu.bkt.clouddn.com/priorityblockingqueue_heap_poll.png)


## remove
```Java
// remove操作结合了上冒操作和下冒操作
public boolean remove(Object o) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = indexOf(o);
        if (i == -1)
            // 元素不存在
            return false;
        removeAt(i);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### indexOf
```Java
// 变量数组，匹配成功，返回索引，否则返回-1
private int indexOf(Object o) {
    if (o != null) {
        Object[] array = queue;
        int n = size;
        for (int i = 0; i < n; i++)
            if (o.equals(array[i]))
                return i;
    }
    return -1;
}
```

### removeAt

```Java
// 先下冒，在上冒（不一定存在）
// 上冒和上冒的过程请参照上面的分析
private void removeAt(int i) {
    Object[] array = queue;
    int n = size - 1;
    if (n == i)
        // 移除最后一个元素
        array[i] = null;
    else {
        E moved = (E) array[n]; // 暂存尾节点
        array[n] = null; // 置空尾节点
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            // 原尾节点从索引i开始下冒过程
            siftDownComparable(i, moved, array, n);
        else
            siftDownUsingComparator(i, moved, array, n, cmp);
        // 这个地方很关键，array[i]==moved说明在下冒过程中，尾节点直接移动到索引为i的节点，
        // 这仅仅只能保证以array[i]为根节点的子树能保证堆的特性，但无法保证以array[0]根的子树能保证堆的特性，
        // 因为array[i]有可能小于父节点array[(i-1)/2]，因此还需要进行一次上冒过程
        // 如果array[i]!=moved，说明moved已经下冒到array[i]的子树中去，
        // 而array[i]是以前子树中的一员必然大于等于父节点array[(i-1)/2]
        if (array[i] == moved) { 
            if (cmp == null)
                // 原尾节点从索引i开始上冒过程
                siftUpComparable(i, moved, array);
            else
                siftUpUsingComparator(i, moved, array, cmp);
        }
    }
    size = n;
}
```


### 逻辑示意图

```Java
public static void main(String[] args) {
    PriorityBlockingQueue<Integer> queue = new PriorityBlockingQueue(15);
    Arrays.asList(0,                        // 第1层
          20, 10,                           // 第2层
          21, 22, 11, 12,                   // 第3层
          23, 24, 25, 26, 13, 14, 15, 16)   // 第4层
          .forEach(i -> queue.add(i));
    queue.remove(22);
}
```
![priorityblockingqueue_heap_remove.png](http://otr5jjzeu.bkt.clouddn.com/priorityblockingqueue_heap_remove.png)
<!-- indicate-the-source -->


