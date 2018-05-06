---
layout:     post
title:      单例模式简介
date:       2018-05-06 23:29:00
categories: 设计模式
tags:       设计模式 单例
---

* 文章目录
{:toc}

> 1. 单例类只能有一个实例。
> 2. 单例类必须自己创建自己的唯一实例。
> 3. 单例类必须给所有其他对象提供这一实例。



#### 懒汉式

```java
public class LazySingleton {
    private LazySingleton singleton = null;

    public LazySingleton getSingleton() {
        if (singleton == null) {
            singleton = new LazySingleton();
        }
        return singleton;
    }
}
```
懒汉式在调用方法时初始化对象，所以是非线程安全的，可以通过下面这三种方式实现线程安全

##### 1.方法加锁

```java
public synchronized LazySingleton getSingleton() {
    if (singleton == null) {
        singleton = new LazySingleton();
    }
    return singleton;
}
```

##### 2.双重锁定检查

```java
public LazySingleton getSingleton() {
    if (singleton == null) {
        synchronized (LazySingleton.class) {
            if (singleton == null) {
                singleton = new LazySingleton();
            }
        }
    }
    return singleton;
}
```

##### 3.静态内部类

```java
private static class LazyHolder {
    public static final LazySingleton SINGLETON = new LazySingleton();
}

public LazySingleton getSingleton() {
    return LazyHolder.SINGLETON;
}
```
通过静态内部类的方式，首先确保延迟加载，任然是懒汉式，其次通过内部类持有的静态对象，确保线程安全，因为不用加锁，比方式1、2都高效


#### 饿汉式


```java
public class StarveSingleton {

    private static StarveSingleton singleton = new StarveSingleton();

    public static StarveSingleton getSingleton() {
        return singleton;
    }
}
```

饿汉式由于拥有一个静态的对象，并在声明时初始化，所以饿汉式天生是线程安全的

#### 枚举
值得注意的是Java中的枚举类型也是单例的，也是线程安全的，而且还自动支持序列化机制，防止反序列化重新创建新的对象，绝对防止多次实例化。