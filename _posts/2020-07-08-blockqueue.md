---
layout: post
title:  "一篇带你了解阻塞队列"
date:   2020-07-08 17:20:18
categories: 并发编程
tags: Java 多线程 阻塞 Queue
---

* content
{:toc}

阻塞队列经常出现在生产者消费者模式中，是并发编程中提高并发能力不可或缺的构件，我们要做的是在不同的场景中选择最适合的阻塞队列。





### 常用的阻塞队列
线程池内部结构主要由四部分组成：线程池管理器负责管理线程池的创建，销毁，添加任务等；工作线程负责从任务队列中获取任务并执行；任务队列作为缓冲机制；任务。

#### 三组添加和移除方法
`BlockingQueue` 中最常用的和添加、删除相关的有8 个方法，把它们分为三组，每组方法都和添加、移除元素相关。区别仅在于特殊情况：当队列满了无法添加元素，或者是队列空了无法移除元素时，不同组的方法对于这种特殊情况会有不同的处理方式。

1. 抛出异常：add、remove、element(返回头节点但不删除)
2. 返回结果但不抛出异常：offer(添加失败会返回false而不是异常)、poll(移除并返回，队列为空会返回Null)、peek
3. 阻塞：put、take


#### LinkedBlockingQueue 
对于 FixedThreadPool 和 SingleThreadExector 而言，它们使用的阻塞队列是容量为 Integer.MAX_VALUE 的 LinkedBlockingQueue，可以认为是无界队列。
#### SynchronousQueue
- 对应的线程池是` CachedThreadPool`。线程池 CachedThreadPool 的最大线程数是 Integer 的最大值，可以理解为线程数是可以无限扩展的。
- SynchronousQueue 最大的不同之处在于，它的容量为 0，所以没有一个地方来暂存元素，导致每次取数据都要先阻塞，直到有数据被放入
#### DelayedWorkQueue
- 它对应的线程池分别是 `ScheduledThreadPool` 和 `SingleThreadScheduledExecutor`，这两种线程池的最大特点就是可以延迟执行任务，比如说一定时间后执行任务或是每隔一定的时间执行一次任务。而延迟队列正好可以把任务按时间进行排序，越靠近队列头代表越早过期。
- 放入的元素必须实现 Delayed 接口，而 Delayed 接口又继承了 Comparable 接口.
#### PriorityBlockingQueue
- PriorityBlockingQueue 是一个支持优先级的无界阻塞队列，可以通过自定义类实现 `compareTo()` 方法来指定元素排序规则，或者初始化时通过构造器参数 `Comparator` 来指定排序规则。同时，插入队列的对象必须是可比较大小的，也就是 实现Comparable 接口的。
- 它的 take 方法在队列为空的时候会阻塞，但是正因为它是无界队列，而且会自动扩容，所以它的队列永远不会满，所以它的 put 方法永远不会阻塞，添加操作始终都会成功，也正因为如此，它的成员变量里只有一个 `Condition`。
```java
private final Condition notEmpty;
```

#### ArrayBlockingQueue
ArrayBlockingQueue 是最典型的有界队列，其内部是用数组存储元素的，利用 `ReentrantLock `实现线程安全。在创建它的时候就需要指定它的容量，之后也不可以再扩容了，在构造函数中我们同样可以指定是否是公平的。
```java
ArrayBlockingQueue(int capacity, boolean fair)
```
##### 安全并发原理
ArrayBlockingQueue 重要的属性
```java
// 用于存放元素的数组
final Object[] items;
// 下一次读取操作的位置
int takeIndex;
// 下一次写入操作的位置
int putIndex;
// 队列中的元素数量
int count;
```
重要的变量
```java
// 以下3个是控制并发用的工具
final ReentrantLock lock;
private final Condition notEmpty;
private final Condition notFull;
```
- `ArrayBlockingQueue` 实现并发同步的原理就是利用` ReentrantLock `和它的两个 `Condition`，读操作和写操作都需要先获取到 ReentrantLock 独占锁才能进行下一步操作。进行读操作时如果队列为空，线程就会进入到读线程专属的 `notEmpty` 的 Condition 的队列中去排队，等待写线程写入新的元素；同理，如果队列已满，这个时候写操作的线程会进入到写线程专属的` notFull` 队列中去排队，等待读线程将队列元素移除并腾出空间。

```java
//put()方法
public void put(Object o) throws InterruptedException {
    lock.lock();
    try {
        while (queue.size() == max) {
        notFull.await();
    }
    queue.add(o);
    notEmpty.signalAll();
    } finally {
        lock.unlock();
    }
}
```
这个方法就是通过Condition实现生产者、消费者模式。

#### 阻塞队列的选择
借鉴线程池选择阻塞队列的思路
- `FixedThreadPool`（SingleThreadExecutor 同理）选取的是 `LinkedBlockingQueue`。 FixedThreadPool 的线程数是固定的，在任务激增的时候，它无法增加更多的线程来帮忙处理 Task，所以需要像 LinkedBlockingQueue 这样没有容量上限的 Queue 来存储那些还没处理的 Task。队列永远不会被填满，就保证了不会拒绝新任务的提交，也不会丢失数据。
- `CachedThreadPool`选取的是`SynchronousQueue`。CachedThreadPool线程的最大数量是无限的，也就意味它不需要一个额外的空间来存储 Task，因为每个任务都可以通过新建线程来处理。
- `ScheduledThreadPool`选取的是延迟队列。ScheduledThreadPool 处理的是基于时间而执行的 Task，而延迟队列有能力把 Task 按照执行时间的先后进行排序。

- 总结：我们可以从五个方面来选择。功能（是否需要排序）、容量、能否扩容、内存结构（数组或链表）、性能。