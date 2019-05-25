---
title: Java并发 -- 安全性、活跃性、性能
date: 2019-04-23 11:00:47
categories:
    - Java Concurrent
tags:
    - Java Concurrent
mathjax: true
---

## 安全性问题
1. 线程安全的本质是**正确性**，而正确性的含义是**程序按照预期执行**
2. 理论上**线程安全**的程序，应该要避免出现**可见性问题（CPU缓存）、原子性问题（线程切换）和有序性问题（编译优化）**
3. 需要分析是否存在线程安全问题的场景：_**存在共享数据且数据会发生变化，即有多个线程会同时读写同一个数据**_
4. 针对该理论的解决方案：不共享数据，采用**线程本地存储**（Thread Local Storage，TLS）；**不变模式**

<!-- more -->

### 数据竞争
数据竞争（**Data Race**）：多个线程**同时访问**同一数据，并且**至少有一个**线程会写这个数据

#### add
```java
private static final int MAX_COUNT = 1_000_000;
private long count = 0;

// 非线程安全
public void add() {
    int index = 0;
    while (++index < MAX_COUNT) {
        count += 1;
    }
}
```

#### add + synchronized
```java
private static final int MAX_COUNT = 1_000_000;
private long count = 0;

public synchronized long getCount() {
    return count;
}

public synchronized void setCount(long count) {
    this.count = count;
}

// 非线程安全
public void add() {
    int index = 0;
    while (++index < MAX_COUNT) {
        setCount(getCount() + 1);
    }
}
```
1. 假设count=0，当两个线程同时执行getCount()，都会返回0
2. 两个线程执行getCount()+1，结果都是1，最终写入内存是1，不符合预期，这种情况为**竟态条件**

### 竟态条件
1. 竟态条件（**Race Condition**）：程序的执行结果依赖于_**线程执行的顺序**_
2. 在并发环境里，线程的执行顺序是不确定的
    - 如果程序存在**竟态条件**问题，那么意味着程序的**执行结果是不确定**的

#### 转账
```java
public class Account {
    private int balance;

    // 非线程安全，存在竟态条件，可能会超额转出
    public void transfer(Account target, int amt) {
        if (balance > amt) {
            balance -= amt;
            target.balance += amt;
        }
    }
}
```

### 解决方案
面对**数据竞争**和**竟态条件**问题，可以通过**互斥**的方案来实现**线程安全**，互斥的方案可以统一归为_**锁**_

## 活跃性问题
活跃性问题：**某个操作无法执行下去**，包括三种情况：_**死锁**、**活锁**、**饥饿**_

### 死锁
1. 发生死锁后线程会**相互等待**，表现为线程_**永久阻塞**_
2. 解决死锁问题的方法是**规避死锁**（破坏发生死锁的条件之一）
    - **互斥**：不可破坏，锁定目的就是为了互斥
    - **占有且等待**：一次性申请**所有**需要的资源
    - **不可抢占**：当线程持有资源A，并尝试持有资源B时失败，线程**主动释放**资源A
    - **循环等待**：将资源编号**排序**，线程申请资源时按**递增**（或递减）的顺序申请

### 活锁
1. 活锁：线程并没有发生阻塞，但由于**相互谦让**，而导致执行不下去
2. 解决方案：在谦让时，尝试**等待一个随机时间**（分布式一致算法Raft也有采用）

### 饥饿
1. 饥饿：线程因**无法访问所需资源**而无法执行下去
    - 线程的**优先级**是不相同的，在CPU繁忙的情况下，优先级低的线程得到执行的机会很少，可能发生线程饥饿
    - 持有锁的线程，如果**执行的时间过长**（持有的资源不释放），也有可能导致饥饿问题
2. 解决方案
    - 保证资源充足
    - 公平地分配资源（**公平锁**） -- 比较可行
    - 避免持有锁的线程长时间执行

## 性能问题
1. 锁的**过度使用**可能会导致**串行化的范围过大**，这会影响多线程优势的发挥（并发程序的目的就是为了**提升性能**）
2. **尽量减少串行**，假设**串行百分比**为5%，那么**多核多线程**相对于**单核单线程**的提升公式（Amdahl定律）
    - $S = \frac{1}{(1-p)+\frac{p}{n}}$，n为CPU核数，p为并行百分比，(1-p)为串行百分比
    - 假如p=95%，n无穷大，加速比S的极限为20，即无论采用什么技术，最高只能提高20倍的性能

### 解决方案
1. **无锁算法和数据结构**
    - 线程本地存储（Thread Local Storage，TLS）
    - 写入时复制（Copy-on-write）
    - 乐观锁
    - JUC中的原子类
    - Disruptor（无锁的内存队列）
2. **减少锁持有的时间**，互斥锁的本质是将并行的程序串行化，要增加并行度，一定要减少持有锁的时间
    - 使用**细粒度锁**，例如JUC中的ConcurrentHashMap（分段锁）
    - 使用**读写锁**，即读是无锁的，只有写才会互斥的

### 性能指标
1. **吞吐量**：在**单位时间**内能处理的请求数量，吞吐量越高，说明性能越好
2. **延迟**：从发出请求到收到响应的时间，延迟越小，说明性能越好
3. **并发量**：能**同时**处理的请求数量，一般来说随着并发量的增加，延迟也会增加，所以**延迟一般是基于并发量来说的**

<!-- indicate-the-source -->