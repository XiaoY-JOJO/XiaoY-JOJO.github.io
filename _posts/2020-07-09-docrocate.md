---
layout: post
title:  "设计模式-装饰者模式"
date:   2020-07-09 16:51:18
categories: 设计模式
tags: Java 编程思想
---

* content
{:toc}

从角色出发，构造装饰器模式示例。





### 装饰器模式
动态地给一个对象增加一些额外的职责，增加对象功能来说，装饰模式比生成子类实现更为灵活。通常会定义一个抽象装饰类，而将具体的装饰类作为它的子类。

#### 角色
1. Component（抽象构件）：它是具体构件和抽象装饰类的共同父类，声明了在具体构件中实现的业务方法
2. ConcreteComponent（具体构件）：它是抽象构件类的子类，用于定义具体的构件对象，实现了在抽象构件中声明的方法，装饰器可以给它增加额外的职责（方法）。
3. Decorator（抽象装饰类）：它也是抽象构件类的子类，用于给具体构件增加职责
4. ConcreteDecorator（具体装饰类）：它是抽象装饰类的子类，负责向构件添加新的职责。每一个具体装饰类都定义了一些新的行为，它可以调用在抽象装饰类中定义的方法，并可以增加新的方法用以扩充对象的行为。

#### 示例
抽象构件
```java
public abstract class ABattercake {
    protected abstract String getDesc();
    protected abstract int cost();
}
```
具体构件
```java
public class Battercake extends ABattercake {
    @Override
    protected String getDesc() {
        return "煎饼";
    }
    @Override
    protected int cost() {
        return 8;
    }
}
```
抽象装饰类，抽象装饰类通过成员属性的方式将抽象构件组合进来，同时也继承了抽象构件，且这里定义了新的业务方法 doSomething()
```java
public abstract class AbstractDecorator extends ABattercake {
    private ABattercake aBattercake;

    public AbstractDecorator(ABattercake aBattercake) {
        this.aBattercake = aBattercake;
    }

    protected abstract void doSomething();

    @Override
    protected String getDesc() {
        return this.aBattercake.getDesc();
    }
    @Override
    protected int cost() {
        return this.aBattercake.cost();
    }
}
```
鸡蛋装饰器，继承了抽象装饰类，鸡蛋装饰器在父类的基础上增加了一个鸡蛋，同时价格加上 1 块钱
```java
public class EggDecorator extends AbstractDecorator {
    public EggDecorator(ABattercake aBattercake) {
        super(aBattercake);
    }

    @Override
    protected void doSomething() {
    }

    @Override
    protected String getDesc() {
        return super.getDesc() + " 加一个鸡蛋";
    }

    @Override
    protected int cost() {
        return super.cost() + 1;
    }

    public void egg() {
        System.out.println("增加了一个鸡蛋");
    }
}

public class Test {
    public static void main(String[] args) {
        ABattercake aBattercake = new Battercake();
        aBattercake = new EggDecorator(aBattercake);
        System.out.println(aBattercake.getDesc() + ", 销售价格: " + aBattercake.cost());
    }
}
```
