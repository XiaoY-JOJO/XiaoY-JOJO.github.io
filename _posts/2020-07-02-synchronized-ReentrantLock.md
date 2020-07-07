---
layout: post
title:  "详解ReentrantLock原理 "
date:   2020-07-03 16:29:11
categories: 线程安全
tags: Java 源码解析 多线程 锁 
---

* content
{:toc}

synchronized和ReentrantLock的区别是什么？ReentrantLock的加锁和释放锁是怎么实现的？本文会一一为你解答。






## synchronized和ReentrantLock 

### 区别
1. synchronized 属于独占式悲观锁，是通过 JVM 隐式实现的，ReentrantLock 是 Lock 的默认实现方式之一，它是基于 AQS（Abstract Queued Synchronizer，队列同步器）实现的，它是 Java 语言提供的 API。
2. 在 Java 中每个对象都隐式包含一个 monitor（监视器）对象，synchronized通过是否持有monitor对象来判断是否有执行权，而ReentrantLock的内部有一个state的状态字段用于表示锁是否被占用，如果是 0 则表示锁未被占用，此时线程就可以把 state 改为 1，并成功获得锁。
3. ReentrantLock 可设置为公平锁，而 synchronized 却不行
4. ReentrantLock 只能修饰代码块，而 synchronized 可以用于修饰方法、修饰代码块等
5. ReentrantLock 需要手动加锁和释放锁，如果忘记释放锁，则会造成资源被永久占用，而 synchronized 无需手动释放锁
6. ReentrantLock可以知道是否成功获得了锁，而 synchronized却不行
7. synchronized一定要按照嵌套的顺序加解锁
8. synchronized为不可中断锁，ReentrantLock为可中断锁

### 相同点
1. 都可以保证可见性，即当一个线程修改了共享变量后，其他线程能够立即得知这个修改。
2. 都拥有可重入的特性。

### synchronized分析
- synchronized 代码块相较于ReentrantLock实际上多了 一个monitorenter 和 两个monitorexit 指令，monitorenter 指令被插入到同步代码块的开始位置，而 monitorexit 需要插入到方法正常结束处和异常处两个地方，这样就可以保证抛异常的情况下也能释放锁。、
- 把执行 monitorenter 理解为加锁，执行 monitorexit 理解为释放锁，每个对象维护着一个记录着被锁次数的计数器，未被锁定的对象的该计数器为 0。
- synchronized修饰的方法会有一个叫作 ACC_SYNCHRONIZED 的 flag 修饰符，来表明它是同步方法。

### ReentrantLock 源码分析
ReentrantLock 是通过 lock() 来获取锁，并通过 unlock() 释放锁

#### 加锁过程
```java
Lock lock = new ReentrantLock();
try {
    // 加锁
    lock.lock();
    //......业务处理
} finally {
    // 释放锁
    lock.unlock();
}
```
ReentrantLock中的lock()是通过sync.lock()实现的，但Sync类中的lock()是一个抽象方法，需要子类NonfairSync或FairSync去实现.

1. lock()
```java
//NonfairSync
final void lock() {
    if (compareAndSetState(0, 1))
        // 将当前线程设置为此锁的持有者
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
//FairSync 
final void lock() {
    acquire(1);
}
```
非公平锁中compareAndSetState()，该方法是尝试将 state 值由0置换为1，如果设置成功的话，则说明当前没有其他线程持有该锁，可直接占用该锁，否则需要通过acquire()排队获取。

2. acquire()
```java
public final void acquire(int arg) {
	// 首先尝试获取锁，如果成功则直接返回
    if (!tryAcquire(arg) && 

	// 如果失败则进行如下操作,acquireQueued()尝试获取到锁，如果成功直接退出
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))

	//如果两者都失败，则调用selfInterrupt()中断当前线程
        selfInterrupt();
}

```
- 尝试获取锁成功的话，tryAcquire(arg)会返回true，则判断语句结果为false，因此直接返回，不再进行后续操作。
- 如果获取锁失败，则调用 addWaiter方法把线程包装成Node对象，同时放入到队列中，acquireQueued方法才会尝试获取锁，如果获取失败，则此节点会被挂起

3. tryAcquire()
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {

        // 公平锁比非公平锁多了一行代码 !hasQueuedPredecessors() ，这是两者的核心区别，判断等待队列是否已经存在线程，若存在，则公平锁的线程不再尝试获取锁

        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) { //尝试获取锁
            setExclusiveOwnerThread(current); // 获取成功，标记被抢占
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc); // set state=state+1
        return true;
    }
    return false;
}

```

- hasQueuedPredecessors()用来查看队列中是否有比它等待时间更久的线程，如果没有，就尝试一下是否能获取到锁，如果获取成功，则标记为已经被占用

4. addWaiter()
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
创建一个入队node为当前线程，Node.EXCLUSIVE 是独占锁,Node.SHARED是共享锁。

5. acquireQueued()
```java
/**
 * 队列中的线程尝试获取锁，失败则会被挂起
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true; // 获取锁是否成功的状态标识
    try {
        boolean interrupted = false; // 线程是否被中断
        for (;;) {
            // 获取前一个节点（前驱节点）
            final Node p = node.predecessor();
            // 当前节点为头节点的下一个节点时，有权尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node); // 获取成功，将当前节点设置为 head 节点
                p.next = null; // 原 head 节点出队，等待被 GC
                failed = false; // 获取成功
                return interrupted;
            }
            // 判断获取锁失败后是否可以挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 线程若被中断，返回 true
                interrupted = true;
        }
    } finally {
	//若获取锁失败，将当前节点置为取消状态
        if (failed)
            cancelAcquire(node);
    }
}

```
- 该方法会使用for(;;)无限循环的方式来尝试获取锁，若获取失败，则调用shouldParkAfterFailedAcquire方法，尝试挂起当前线程;
- acquireQueue()完成了两件事情：一是如果当前节点的前驱节点是头节点并且能够获得锁，当前线程成功获取到锁并退出；二是如果获取锁失败，就将当前线程设为取消状态，并阻塞当前线程。

#### 释放锁过程
```java
public void unlock() {
    sync.release(1);
}
public final boolean release(int arg) {
    // 尝试释放锁
    if (tryRelease(arg)) {
        // 释放成功
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

```
先调用 tryRelease 方法尝试释放锁，如果释放成功，则查看头结点的状态是否为 SIGNAL，如果是，则唤醒头结点的下个节点关联的线程；如果释放锁失败，则返回 false。

```java
/**
 * 尝试释放当前线程占有的锁
 */
protected final boolean tryRelease(int releases) {
    int c = getState() - releases; // 释放锁后的状态，0 表示释放锁成功
    // 如果拥有锁的线程不是当前线程的话抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // 锁被成功释放
        free = true;
        setExclusiveOwnerThread(null); // 清空独占线程
    }
    setState(c); // 更新 state 值，0 表示为释放锁成功
    return free;
}
```
在 tryRelease 方法中，会先判断当前的线程是不是占用锁的线程，如果不是的话，则会抛出异常；如果是的话，则先计算锁的状态值 getState() - releases 是否为 0，如果为 0，则表示可以正常的释放锁，然后清空独占的线程，最后会更新锁的状态并返回执行结果。

### 两者如何选择
1. 如果能不用最好既不使用 Lock 也不使用 synchronized，推荐优先使用工具类来加解锁。
2. 如果 synchronized 关键字适合你的程序， 那么请尽量使用它，这样可以减少编写代码的数量，减少出错的概率。
3. 如果特别需要 Lock 的特殊功能，比如尝试获取锁、可中断、超时功能等，才使用 Lock。