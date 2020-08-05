---
layout: post
title:  "ThreadLocal怎么用？"
date:   2020-07-09 17:20:18
categories: 线程安全
tags: Java 多线程 
---

* content
{:toc}

从底层原理分析ThreadLocal如何解决线程安全问题，同sychronized有什么区别？究竟如何使用？





## ThreadLocal
- `ThreadLocal` 解决线程安全问题的时候，相比于使用“锁”而言，换了一个思路，把资源变成了各线程独享的资源，非常巧妙地避免了同步操作。具体而言，它可以在 `initialValue` 中 new 出自己线程独享的资源，而多个线程之间，它们所访问的对象本身是不共享的，自然就不存在任何并发问题。这是 ThreadLocal 解决并发问题的最主要思路。
- 如果我们把放到 ThreadLocal中的`资源`用static修饰，让它变成一个共享资源的话，那么即便使用了 ThreadLocal，同样也会有线程安全问题。
- ThreadLocal为什么用`static`修饰？非静态变量是对象所拥有的，在创建对象的时候被初始化，也就是说，在一个线程内，没有被static修饰的ThreadLocal变量实例，会随着所在的类多次创建而被多次实例化，虽然ThreadLocal限制了变量的作用域，但这样频繁的创建变量实例是没有必要的。


### 应用场景
#### 保存每个线程独享的对象
ThreadLocal用作保存每个线程独享的对象，为每个线程都创建一个副本，这样每个线程都可以修改自己所拥有的副本，而不会影响其他线程的副本，确保了线程安全。

```java
public class ThreadLocalDemo06 {

	public static ExecutorService threadPool = Executors.newFixedThreadPool(16);

	public static void main(String[] args) throws InterruptedException {
            for (int i = 0; i < 1000; i++) {
                int finalI = i;
                threadPool.submit(new Runnable() {
                    @Override
                    public void run() {
                        String date = new ThreadLocalDemo06().date(finalI);
                    System.out.println(date);
                    }
                });
            }
        threadPool.shutdown();
	}

    public String date(int seconds) {
        Date date = new Date(1000 * seconds);
        SimpleDateFormat dateFormat = ThreadSafeFormatter.dateFormatThreadLocal.get();
        return dateFormat.format(date);
    }
}

class ThreadSafeFormatter {
    public static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal = new ThreadLocal<SimpleDateFormat>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("mm:ss");
        }
    };
}

```
使用了 ThreadLocal 帮每个线程去生成它自己的 simpleDateFormat 对象，对于每个线程而言，对象是独享的，因此这个对象只会创建16个。

#### 每个线程内需要独立保存信息
用 `ThreadLocal` 保存一些业务内容（用户权限信息、从用户系统获取到的用户名、用户ID 等），这些信息在同一个线程内相同，但是不同的线程使用的业务内容是不相同的。通过 ThreadLocal 直接获取到，避免了传参，类似于全局变量的概念。

![](/images/threadlocal1.png)

当一个请求进来的时候，一个线程会负责执行这个请求，然后这个请求就会依次调用 service-1()、service-2()、service-3()、service-4()，这 4 个方法可能是分布在不同的类中的。在 service-1() 的时候它会创建一个 user 的对象，用于保存比如说这个用户的用户名等信息，后面 service-2/3/4() 都需要用到这个对象的信息，比如说 service-2() 代表下订单、service-3() 代表发货、service-4() 代表完结订单，在这种情况下，每一个方法都需要用户信息，所以就需要把这个 user 对象层层传递下去，从 service-1() 传到 service-2()，再从 service-2() 传到 service-3()，以此类推，这样做会导致代码非常冗余。

![](/images/threadlocal2.png)

在这个图中可以看出，同样是多个线程同时去执行，但是这些线程同时去访问这个 ThreadLocal 并且能利用 ThreadLocal 拿到只属于自己的独享对象。这样的话，就无需任何额外的措施，保证了线程安全，因为每个线程是独享 user 对象的。

```java
public class ThreadLocalDemo07 {

    public static void main(String[] args) {
        new Service1().service1();
    }
}

class Service1 {

    public void service1() {
        User user = new User("拉勾教育");
        UserContextHolder.holder.set(user);
        new Service2().service2();
    }
}

class Service2 {

    public void service2() {
        User user = UserContextHolder.holder.get();
        System.out.println("Service2拿到用户名：" + user.name);
        new Service3().service3();
    }
}

class Service3 {

    public void service3() {
        User user = UserContextHolder.holder.get();
        System.out.println("Service3拿到用户名：" + user.name);
        UserContextHolder.holder.remove();
    }
}

class UserContextHolder {

    public static ThreadLocal<User> holder = new ThreadLocal<>();
}

class User {

    String name;

    public User(String name) {
        this.name = n
    }
}

```
- 通过`set()`设置`ThreadLocal`变量，通过`get()`获取该变量
- 通过上述两个例子可以看出`ThreadLocal`一般定义在单独的一个类里
- 还可以用于事务操作中存取事务信息
- 数据库连接，`session`会话管理

### Thread、ThreadLocal及ThreadLocalMap

![](/images/threadlocal.png)

每个`Thread`对象都持有一个`ThreadLocalMap`成员变量，每个ThreadLocalMap存储了多个`ThreadLocal`键值对。

### 源码解析
#### get()
```java 
public T get() {
    //获取到当前线程
    Thread t = Thread.currentThread();
    //获取到当前线程内的 ThreadLocalMap 对象，每个线程内都有一个 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //获取ThreadLocalMap中的Entry对象并拿到 Value,这个Entry是一个内部类
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果线程内之前没创建过 ThreadLocalMap，就创建
    return setInitialValue();
}
```
在 `ThreadLocalMap` 中会有一个 `Entry` 类型的数组，名字叫 table。我们可以把 Entry 理解为一个 map
```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;


        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
   private Entry[] table;
//...
}
```

#### set()
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
		//this为ThreadLocal的引用
    else
        createMap(t, value);
}
```

### 内存泄漏
内存泄漏指的是，当某一个对象不再有用的时候，占用的内存却不能被回收，这就叫作内存泄漏。

#### key的泄露
我们可能会在业务代码中执行了 `ThreadLocal instance = null` 操作，想清理掉这个 `ThreadLocal` 实例，但是假设我们在`ThreadLocalMap`的 `Entry` 中强引用了 ThreadLocal 实例，那么，虽然在业务代码中把 ThreadLocal 实例置为了 null，但是在Thread类中依然有这个引用链的存在。GC在垃圾回收的时候会进行可达性分析，它会发现这个 ThreadLocal 对象依然是可达的。

#### value的泄露
#### 如何预防
调用 ThreadLocal 的 `remove`方法。调用这个方法就可以删除对应的 `value` 对象，可以避免内存泄漏。
