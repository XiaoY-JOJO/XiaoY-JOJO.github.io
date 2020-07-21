---
layout: post
title:  "String源码解析"
date:   2020-07-02 16:34:05
categories: Java源码解析
tags: Java 源码解析 String 
---

* content
{:toc}

本文主要讲解String源码，包括其中较为重要的方法，设计思路，以及扩展知识点。





## String

### 源码解析
```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
	// 用于存储字符串的值
	private final char value[];
	// 缓存字符串的 hash code
	private int hash; 
}
```
可以看出`String`内部实际存储结构为一个由`final`修饰的`char数组`，因此该数组的值不可改变

#### 构造方法
```java
//参数为string时，将该string的值和hash直接赋给新的String对象
public String(String original) {
	this.value = original.value;
	this.hash = original.hash;
}

//传入char数组类型的参数时，调用Arrays的copyOf将数组的值赋给新对象
public String(char value[]) {
	this.value = Arrays.copyOf(value, value.length);
}

//StringBuffer和StringBuilder类型参数都是先获取对象的值和长度，然后通过copyOf传给新对象
public String(StringBuffer buffer) {
	synchronized(buffer) {
		this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
	}
}

public String(StringBuilder builder) {
	this.value = Arrays.copyOf(builder.getValue(), builder.length());
}

```

#### 常用方法
##### equals()
```java
public boolean equals(Object anObject) {

    if (this == anObject) {
        return true;
    }

    if (anObject instanceof String) {
        String anotherString = (String) anObject;
        int n = value.length;

        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                        return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```
`equals()`方法的核心思路即，①先判断传入的参数是否引用了同一个对象，②如果不是，再判断是否为String类型的对象，③满足该条件时，继续判断两个对象的char数组长度是否一致，④满足条件则循环判断每个对应的元素是否相同。

注意：`equals()`方法传入的参数是Object类型，而不是String.

##### compareTo()
```java
public int compareTo(String anotherString) {
	int len1 = value.length;
	int len2 = anotherString.value.length;
	// 获取到两个字符串长度最短的那个 int 值
	int lim = Math.min(len1, len2);
	char v1[] = value;
	char v2[] = anotherString.value;
	int k = 0;
	// 对比每一个字符
	while (k < lim) {
		char c1 = v1[k];
		char c2 = v2[k];
		if (c1 != c2) {
			// 有字符不相等就返回差值
			return c1 - c2;
		}
		k++;
	}
	return len1 - len2;
}
```
`compareTo()`会直接循环对比每一个元素，只要有字符不相等就返回两个字符的差值，因此只有方法返回0时表示两个String是相等的，`compareTo()`的参数只能是String类型。

##### hashCode()
```java
public int hashCode() {
    int h = hash;
    //如果hash没有被计算过，并且字符串不为空，则进行hashCode计算
    if (h == 0 && value.length > 0) {
        char val[] = value;
        
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
} 
```
可以看出String对象的hashCode其实就是char的累加。

##### concat()

```java
//字符串拼接
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;  
    char buf[] = Arrays.copyOf(value, len + otherLen);
    //buf现在只存了this.value,通过getChars()将strvalue复制到buf数组尾部
    str.getChars(buf, len);
    return new String(buf, true);
}
```

拼接的核心是`getChars()`，而`getChars()`调用的是`System.arraycopy()`，该方法思路如下：①创建一个新长度的数组;
②把旧数组的内容拷贝到新数组;
③再把新拼接的内容拷贝到新数组的后面;
④旧数组被gc掉;
⑤把最终的新数组，当做参数来创建一个新的String。

### 知识扩展
1. == 和 equals 的区别
== 对于基本数据类型来说，是用于比较 “值”是否相等的；而对于引用类型来说，是用于比较引用地址是否相同的，Object 中的 `equals()` 方法其实就是 ==，而 String 重写了 `equals()` 方法把它修改成比较两个字符串的值是否相等。

2. 为什么用final修饰？
- 字符串常量池的需要
- 允许String对象缓存HashCode：字符串不变性保证了hash码的唯一性,因此可以放心地进行缓存.
- 安全性：String被许多的Java类(库)用来当做参数,例如 网络连接地址URL,文件路径path,假若String不是固定不变的,将会引起各种安全隐患。
- 线程安全：同一个字符串实例可以被多个线程共享，这样便不用因为线程安全问题而使用同步，字符串自己便是线程安全的。

3. String 和 StringBuilder、StringBuffer 的区别
- `String`和`StringBuilder`、`StringBuffer`底层都是char[]数组，但是String是final修饰的
- 字符串拼接的时候，都是通过`System.arraycopy`去操作char[]数组，但是StringBuilder、StringBuffer 则是创建一个新的数组，引用没有改变；而String则是另外`new String()`，引用改变了。
- StringBuilder、StringBuffer 提供了`append`和`insert`方法对字符串进行拼接
- StringBuffer使用了 `synchronized`来保证线程安全，所以性能不是很高

4. String和JVM
- 直接定义的String a = "a" 是储存在 常量存储区中的字符串常量池中；new String("a")是存储在堆中；
- 常量池中相同的字符串只会有一个，但是new String()，每new一个对象就会在堆中新建一个对象，不管这个值是否相同
- JDK 1.7 之后把永生代换成的元空间，把字符串常量池从方法区移到了 Java 堆上

### 思考
借鉴String类的设计，自己实现一个不可变类。
1. 将类声明为`final`，所以它不能被继承。
2. 将所有的成员声明为私有的，这样就不允许直接访问这些成员。
3. 对变量不要提供setter方法。
4. 将所有可变的成员声明为final，这样只能对它们赋值一次。
5. 通过构造器初始化所有成员，进行深拷贝(deep copy)。
6. 在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝。
