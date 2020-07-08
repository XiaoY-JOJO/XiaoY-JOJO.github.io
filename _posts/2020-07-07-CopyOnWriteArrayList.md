---
layout: post
title:  "冷门但重要的容器-CopyOnWriteArrayList "
date:   2020-07-07 17:27:11
categories: 数据结构
tags: Java 多线程 JUC 
---

* content
{:toc}

CopyOnWriteArrayList好像很“冷门”，但它的应用其实很广泛，是一个线程安全的并发容器，也是很可能被深挖的知识点。





## CopyOnWriteArrayList是什么？
线程安全的`Vector` 内部是使用 synchronized 来保证线程安全的，并且锁的粒度比较大，都是方法级别的锁，在并发量高的时候，很容易发生竞争，并发效率相对比较低。从 JDK1.5 开始，Java 并发包里提供了使用 `CopyOnWrite` 机制实现的并发容器  `CopyOnWriteArrayList` 作为主要的并发 List。

### 适用场景
1. 读操作尽可能快，写可以慢一些
2. 读多写少

### 读写规则
- `CopyOnWriteArrayList` 比读写锁的思想更进一步，CopyOnWriteArrayList读取是完全不用加锁的，写入也不会阻塞读取操作，也就是说你可以在写入的同时进行读取，只有写入和写入之间需要进行同步。这样一来，读操作的性能就会大幅度提升。

### 特点
1. `CopyOnWrite`的含义：当容器需要被修改的时候，不直接修改当前容器，而是先将当前容器进行 Copy，复制出一个新的容器，然后修改新的容器，完成修改之后，再将原容器的引用指向新的容器。因此可以对 CopyOnWrite 容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素，也不会有修改。 CopyOnWrite 容器也是一种读写分离的思想体现，读和写使用不同的容器。

2. 迭代期间可以修改容器内容

```java
/**
* 描述： 演示CopyOnWriteArrayList迭代期间可以修改集合的内容
*/
public class CopyOnWriteArrayListDemo {
    public static void main(String[] args) {
 
        CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>(new Integer[]{1, 2, 3});
 
        System.out.println(list); //[1, 2, 3]
 
        //Get iterator 1
        Iterator<Integer> itr1 = list.iterator();
 
        //Add one element and verify list is updated
        list.add(4);
 
        System.out.println(list); //[1, 2, 3, 4]
 
        //Get iterator 2
        Iterator<Integer> itr2 = list.iterator();
 
        System.out.println("====Verify Iterator 1 content====");
 
        itr1.forEachRemaining(System.out::println); //1,2,3
 
        System.out.println("====Verify Iterator 2 content====");
 
        itr2.forEachRemaining(System.out::println); //1,2,3,4
 
    } 
}
```

`CopyOnWriteArrayList`的迭代器一旦被建立之后，如果往之前的 CopyOnWriteArrayList对象中去新增元素，在迭代器中既不会显示出元素的变更情况，同时也不会报错。

### 缺点
- 因为 CopyOnWrite 的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，这一点会占用额外的内存空间
- 在元素较多或者复杂的情况下，复制的开销很大
-  CopyOnWrite 容器的修改是先修改副本，所以这次修改对于其他线程来说，并不是实时能看到的

### 源码解析
#### 数据结构

```java
//可重入锁对象，用来保证修改操作的线程安全 
final transient ReentrantLock lock = new ReentrantLock();
 
// CopyOnWriteArrayList底层由数组实现，volatile修饰，保证数组的可见性 
private transient volatile Object[] array;
 
final Object[] getArray() {
    return array;
}
 
//将原容器的引用指向新容器
final void setArray(Object[] a) {
    array = a;
}

//初始化CopyOnWriteArrayList相当于初始化数组
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```

#### add()

```java
public boolean add(E e) {
    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
 
        // 得到原数组的长度和元素
        Object[] elements = getArray();
        int len = elements.length;
 
        // 复制出一个新数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
 
        // 添加时，将新元素添加到新数组中
        newElements[len] = e;
 
        // 将volatile Object[] array 的指向替换成新数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

#### set()
```java
public E set(int index, E element) { 
	final ReentrantLock lock = this.lock;
 	lock.lock();
 	try { 
		// 得到原数组的旧值 
		Object[] elements = getArray();
 		E oldValue = get(elements, index);
 		// 判断新值和旧值是否相等
 		if (oldValue != element) { 
			// 复制新数组，新值在新数组中完成 
			int len = elements.length;
 			Object[] newElements = Arrays.copyOf(elements, len); 
			newElements[index] = element; 
			// 将array引用指向新数组 setArray(newElements);
 		} else {
 			setArray(elements); 
		} 
		return oldValue;
 	}finally { 
		lock.unlock();
 	}
} 
```
思路与`add()`方法类似

#### get()
```java
//一共包含了两个重载和getArray，都没有加锁
public E get(int index) {
    return get(getArray(), index);
}
final Object[] getArray() {
    return array;
}
private E get(Object[] a, int index) {
    return (E) a[index];
}

```
#### 迭代器 COWIterator 类

```java
private COWIterator(Object[] elements, int initialCursor) {
    cursor = initialCursor;
    snapshot = elements;
}
//迭代器在被构建的时候，会把当时的 elements 赋值给 snapshot，而之后的迭代器所有的操作都基于 snapshot 数组进行的

//比如next()
public E next() {
    if (! hasNext())
        throw new NoSuchElementException();
    return (E) snapshot[cursor++];
}

```
返回的内容是 snapshot 对象，所以，后续就算原数组被修改，这个 snapshot 既不会感知到，也不会受影响，执行迭代操作不需要加锁，也不会因此抛出异常。迭代器返回的结果，和创建迭代器的时候的内容一致。解释了为什么迭代时可以修改容器。