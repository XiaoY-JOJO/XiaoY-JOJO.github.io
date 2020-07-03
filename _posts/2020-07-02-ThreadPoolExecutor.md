---
layout: post
title:  "带你解析ThreadPoolExecutor最底层 "
date:   2020-07-03 16:50:41
categories: 并发编程
tags: Java 源码解析 线程池
---

* content
{:toc}

从ThreadPoolExecutor的核心参数，四种拒绝策略，到执行原理，让我们一块来探索ThreadPoolExecutor！






## ThreadPoolExecutor

使用Executors创建线程池的弊端：
1. FixedThreadPool 和 SingleThreadPool：允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM
2. CachedThreadPool 和 ScheduledThreadPool：允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM

### 核心参数
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {

}
```

1. corePoolSize 表示线程池的常驻核心线程数。如果设置为0，则表示在没有任何任务时，销毁线程池；如果大于 0，即使没有任务时也会保证线程池的线程数量等于此值。此值如果设置的比较小，则会频繁的创建和销毁线程；如果设置的比较大，则会浪费系统资源。

2. maximumPoolSize 表示线程池在任务最多时，最大可以创建的线程数。官方规定此值必须大于 0，也必须大于等于 corePoolSize，此值只有在任务比较多，且不能存放在任务队列时，才会用到。

3. keepAliveTime 表示线程的存活时间，当线程池空闲时并且超过了此时间，多余的线程就会销毁，直到线程池中的线程数量销毁的等于 corePoolSize 为止。

4. unit 表示存活时间的单位，它是配合 keepAliveTime 参数共同使用的。

5. workQueue 表示线程池执行的任务队列，当线程池的所有线程都在处理任务时，如果来了新任务就会缓存到此任务队列中排队等待执行。

6. threadFactory 表示线程的创建工厂，通常在创建线程池时不指定此参数.
7. RejectedExecutionHandler 表示指定线程池的拒绝策略，当线程池的任务已经在缓存队列 workQueue 中存储满了之后，并且不能创建新的线程来执行此任务时，就会用到此拒绝策略，它属于一种限流保护的机制。

### 线程池的工作流程
- 任务添加进线程池后，①先判断是否有空线程，决定是否执行；②然后判断是否可存入队列，决定是否存入；③最后判断是否超过最大线程数，决定创建线程执行任务，还是执行拒绝策略。
- 任务的执行：execute()-->addWorker()-->runworker(getTask)

#### 四种拒绝策略
1. AbortPolicy，这种拒绝策略在拒绝任务时，会直接抛出一个类型为 RejectedExecutionException 的 RuntimeException，让你感知到任务被拒绝了，于是你便可以根据业务逻辑选择重试或者放弃提交等策略。
2. DiscardPolicy，这种拒绝策略正如它的名字所描述的一样，当新任务被提交后直接被丢弃掉，也不会给你任何的通知，相对而言存在一定的风险，因为我们提交的时候根本不知道这个任务会被丢弃，可能造成数据丢失。
3. DiscardOldestPolicy，如果线程池没被关闭且没有能力执行，则会丢弃任务队列中的头结点，通常是存活时间最长的任务，这种策略与第二种不同之处在于它丢弃的不是最新提交的，而是队列中存活时间最长的，这样就可以腾出空间给新提交的任务，但同理它也存在一定的数据丢失风险。
4. CallerRunsPolicy，相对而言它就比较完善了，当有新任务提交后，如果线程池没被关闭且没有能力执行，则把这个任务交于提交任务的线程执行，也就是谁提交任务，谁就负责执行任务。

#### 执行方法execute()
```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 当前工作的线程数小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        // 创建新的线程执行此任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 核心线程已满，检查线程池是否处于运行状态，以及是否能将任务添加到队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查线程池是否处于运行状态，防止在第一次校验通过后线程池关闭
        // 如果是非运行状态，则将刚加入队列的任务移除
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果线程池的线程数为 0 时（当 corePoolSize 设置为 0 时会发生）
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false); // 新建非核心线程执行任务
    }
    // 核心线程都在忙，此时判断当前线程数是否大于最大线程数，若大于则创建新的非核心线程
    else if (!addWorker(command, false)) 
        // 创建失败则执行拒绝策略
        reject(command);
}

```
- execute()方法包含了线程池的整个工作流程，包括是否创建线程并执行任务，是否存入队列，是否执行拒绝策略。
- 为什么要double check线程池的状态？
- 在多线程环境下，线程池的状态时刻在变化，而ctl.get()是非原子操作，很有可能刚获取了线程池状态后线程池状态就改变了。判断是否将command加入workque是线程池之前的状态。倘若没有double check，万一线程池处于非running状态（在多线程环境下很有可能发生），那么command永远不会执行。

#### addWorker()
addWorker()主要负责创建新线程并执行任务、
```
private boolean addWorker(Runnable firstTask, boolean core) {
       // CAS更新线程池数量
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //当core为false,创建非核心线程，此时判断当前线程数是否大于maximumPoolSize
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        	//创建worker对象及Thread对象
            w = new Worker(firstTask);
            final Thread t = w.thread;
            //启动该Thread，便会触发Worker的run()方法被调用
            if (t != null) {
                // 线程池重入锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();  // 线程启动，执行任务（Worker.thread(firstTask).start()）;
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
addWorker(Runnable firstTask, boolean core)的两个参数，firstTask表示任务，core：是否可以创建线程的阈值，为true表示使用 corePoolSize 作为阀值，false 则表示使用 maximumPoolSize 作为阀值。

#### Worker类
```
 private final class Worker
         extends AbstractQueuedSynchronizer
         implements Runnable{
     Worker(Runnable firstTask) {
         setState(-1); // inhibit interrupts until runWorker
         this.firstTask = firstTask;
         this.thread = getThreadFactory().newThread(this); // 创建线程
     }
     /** Delegates main run loop to outer runWorker  */
     public void run() {
         runWorker(this);
     }
     // ...
 }
```
从Woker类的构造方法实现可以发现：线程工厂在创建线程thread时，将Woker实例本身this作为参数传入，当执行start方法启动线程thread时，本质是执行了Worker的runWorker方法。

```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 将worker的状态重置为正常状态，因为state状态值在构造器中被初始化为-1
    w.unlock();
    // 通过completedAbruptly变量的值判断任务是否正常执行完成
    boolean completedAbruptly = true;
    try {
        // 如果task为null就通过getTask方法获取阻塞队列中的下一个任务
        // getTask方法一般不会返回null，所以这个while类似于一个无限循环
        // worker对象就通过这个方法的持续运行来不断处理新的任务
        while (task != null || (task = getTask()) != null) {
            // 每一次任务的执行都必须获取锁来保证下方临界区代码的线程安全
            w.lock();

            // 如果状态值大于等于STOP（状态值是有序的，即STOP、TIDYING、TERMINATED）
            // 且当前线程还没有被中断，则主动中断线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();

            // 开始
            try {
                // 执行任务前处理操作，默认是一个空实现
                // 在子类中可以通过重写来改变任务执行前的处理行为
                beforeExecute(wt, task);

                // 通过thrown变量保存任务执行过程中抛出的异常
                // 提供给下面finally块中的afterExecute方法使用
                Throwable thrown = null;
                try {
                    // *** 重要：实际执行任务的代码
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    // 因为Runnable接口的run方法中不能抛出Throwable对象
                    // 所以要包装成Error对象抛出
                    thrown = x; throw new Error(x);
                } finally {
                    // 执行任务后处理操作，默认是一个空实现
                    // 在子类中可以通过重写来改变任务执行后的处理行为
                    afterExecute(task, thrown);
                }
            } finally {
                // 将循环变量task设置为null，表示已处理完成
                task = null;
                // 累加当前worker已经完成的任务数
                w.completedTasks++;
                // 释放while体中第一行获取的锁
                w.unlock();
            }
        }

        // 将completedAbruptly变量设置为false，表示任务正常处理完成
        completedAbruptly = false;
    } finally {
        // 销毁当前的worker对象，并完成一些诸如完成任务数量统计之类的辅助性工作
        // 在线程池当前状态小于STOP的情况下会创建一个新的worker来替换被销毁的worker
        processWorkerExit(w, completedAbruptly);
    }
}
```

runWorker方法是线程池执行任务的核心：
1. 线程启动之后，通过unlock方法释放锁，设置AQS的state为0，表示运行可中断
2. Worker执行firstTask或从workQueue中获取任务
3. 进行加锁操作，保证thread不被其他线程中断（除非线程池被中断）
4. 检查线程池状态，倘若线程池处于中断状态，当前线程将被中断。
5. 执行beforeExecute
6. 执行任务的run方法
7. 执行afterExecute方法
8. 解锁操作，然后循环执行2-8

#### submit()
```
// submit方法在AbstractExecutorService中的实现
public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();      
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
```
通过submit方法提交的Callable任务会被封装成了一个FutureTask对象。通过Executor.execute方法提交FutureTask到线程池中等待被执行，最终执行的是FutureTask的run方法.

#### 拓展
1. execute()和submit()的区别
- execute()和submit()都是用来执行线程池任务的，它们最主要的区别是，submit()方法可以配合Future类的future.get()接收线程池执行的返回值，而execute()不能接收返回值。
- execute() 方法属于 Executor 接口的方法，而 submit() 方法则是属于 ExecutorService 接口的方法

2. hreadPoolExecutor扩展
- ThreadPoolExecutor 的扩展主要是通过重写它的 beforeExecute() 和 afterExecute() 方法实现的，我们可以在扩展方法中添加日志或者实现数据统计，比如统计线程的执行时间。