---
layout: post
title:  "面试必备-ConcurrentHashMap "
date:   2020-07-07 17:24:11
categories: 数据结构
tags: Java HashMap 多线程 集合 
---

* content
{:toc}

ConcurrentHashMap的原理，与HashMap、HashTable的区别，ConcurrentHashMap的演进和优化，作为面试必备知识点，本文统一为你讲解。






## ConcurrentHashMap
### JDK 7
- 在 `ConcurrentHashMap` 内部进行了 Segment 分段，`Segment` 继承了`ReentrantLock`，可以理解为一把锁，各个Segment之间都是相互独立上锁的，互不影响。
- 每个`Segment`的底层数据结构与`HashMap`类似，仍然是数组和链表组成的拉链法结构。默认有 0~15 共 16 个 Segment，所以最多可以同时支持 16 个线程并发操作（操作分别分布在不同的 Segment 上）。16 这个默认值可以在初始化的时候设置为其他值，但是一旦确认初始化以后，是不可以扩容的。

### JDK 8
- 结构变成了`数组+链表+红黑树`，数组+链表的部分与JDK7版本相同，都是采用拉链发解决哈希冲突，当链表的长度超过一个阈值时，链表结构便会转换为红黑树结构，这样查找的时间复杂度由O(n)变成了O(logn)。
- 链表长度超过 8 就转为红黑树的设计，更多的是为了防止用户自己实现了不好的哈希算法时导致链表过长，从而导致查询效率低，而此时转为红黑树更多的是一种保底策略，用来保证极端情况下查询的效率。

#### 重要源码
##### put()

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) {
        throw new NullPointerException();
    }
    //计算 hash 值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K, V>[] tab = table; ; ) {
        Node<K, V> f;
        int n, i, fh;

        //如果数组是空的，就进行初始化
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();
        }
        // 找该 hash 值对应的数组下标
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //如果该位置是空的，就用 CAS 的方式放入新值
            if (casTabAt(tab, i, null,
                    new Node<K, V>(hash, key, value, null))) {
                break;
            }
        }
        //hash值等于 MOVED 代表在扩容
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
        }
        //槽点上是有值的情况
        else {
            V oldVal = null;

            //用 synchronized 锁住当前槽点，保证并发安全
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //如果是链表的形式
                    if (fh >= 0) {
                        binCount = 1;
                        //遍历链表
                        for (Node<K, V> e = f; ; ++binCount) {
                            K ek;
                            //如果发现该 key 已存在，就判断是否需要进行覆盖，然后返回
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) {
                                    e.val = value;
                                }
                                break;
                            }
                            Node<K, V> pred = e;

                            //到了链表的尾部也没有发现该 key，说明之前不存在，就把新值添加到链表的最后
                            if ((e = e.next) == null) {
                                pred.next = new Node<K, V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    //如果是红黑树的形式
                    else if (f instanceof TreeBin) {
                        Node<K, V> p;
                        binCount = 2;
                        //调用 putTreeVal 方法往红黑树里增加数据
                        if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) {
                                p.val = value;
                            }
                        }
                    }
                }
            }
            if (binCount != 0) {
                //检查是否满足条件并把链表转换为红黑树的形式，默认的 TREEIFY_THRESHOLD 阈值是 8
                if (binCount >= TREEIFY_THRESHOLD) {
                    treeifyBin(tab, i);
                }
                //putVal 的返回是添加前的旧值，所以返回 oldVal
                if (oldVal != null) {
                    return oldVal;
                }
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

```

##### get()
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //计算 hash 值
    int h = spread(key.hashCode());
    //如果整个数组是空的，或者当前槽点的数据是空的，说明 key 对应的 value 不存在，直接返回 null
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        //判断头结点是否就是我们需要的节点，如果是则直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //如果头结点 hash 值小于 0，说明是红黑树或者正在扩容，就用对应的 find 方法来查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历链表来查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}

```

### 两个版本比较
- 数据结构：Java 7 采用 `Segment` 分段锁来实现，而 Java 8 中的 ConcurrentHashMap 使用`数组 + 链表 + 红黑树`。
- 并发能力：ava 7 中，每个 Segment 独立加锁，最大并发个数就是 Segment 的个数，默认是 16；Java 8 中，锁粒度更细，理想情况下 table 数组元素的个数（也就是数组长度）就是其支持并发的最大个数，并发度比之前有提高。
- 并发安全：Java 7 采用 `Segment` 分段锁来保证安全，而 Segment 是继承自 ReentrantLock；Java 8 采用 `Node + CAS + synchronized`保证线程安全。
