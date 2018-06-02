---
layout:     post
title:      享元模式
date:       2018-06-02 16:19:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★★★☆，使用频率：★☆☆☆☆】

&emsp;&emsp;首先解释一下**享元**的概念：在一个系统中如果有多个相同的对象，那么只共享一份就可以了，不必每个都去实例化一个对象，这个共享的对象即为享元。




&emsp;&emsp;比如说一个文本系统，每个字母定一个对象，那么大小写字母一共就是52个，那么就要定义52个对象。如果有一个1M的文本，那么字母是何其的多，如果每个字母都定义一个对象那么内存早就爆了。那么如果要是每个字母都共享一个对象，那么就大大节约了资源。

&emsp;&emsp;在Flyweight模式中，由于要产生各种各样的对象，所以在Flyweight(享元)模式中常出现Factory模式。Flyweight的内部状态是用来共享的,Flyweight factory负责维护一个对象存储池（Flyweight Pool）来存放内部状态的对象。Flyweight模式是一个提高程序效率和性能的模式,会大大加快程序的运行速度。

#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/flyweight/flyweight_uml.png)
在享元模式中，经常会用到工厂来实例化对象，ConcreteFlyweight即为实际共享的对象，享元可以是任何对象。下面看下享元模式的具体代码：

#### 具体代码

---

享元角色：

```java
public abstract class Flyweight {
    public abstract void operation();
}

public class ConcreteFlyweight extends Flyweight {

    private String str;//假设这里的享元是一个字符串

    public ConcreteFlyweight(String str) {
        this.str = str;
    }

    @Override
    public void operation() {
        System.out.println("Concrete flyweight: " + str);
    }
}
```

享元工厂：

```java
public class FlyweightFactory {
    private Map<String, Flyweight> flyweights = new HashMap<String, Flyweight>();

    public FlyweightFactory() {
    }

    public Flyweight getFlyweight(String str) {
        Flyweight flyweight = flyweights.get(str);
        if (flyweight == null) {
            flyweight = new ConcreteFlyweight(str);
            flyweights.put(str, flyweight);
        }
        return flyweight;
    }

    public int getFlyweightSize() {
        return flyweights.size();
    }
}
```

测试类Client:

```java
public class Client {
    public static void main(String[] args) {
        FlyweightFactory factory = new FlyweightFactory();
        Flyweight fw1, fw2, fw3, fw4, fw5, fw6;

        fw1 = factory.getFlyweight("Google");
        fw2 = factory.getFlyweight("google");
        fw3 = factory.getFlyweight("Google");
        fw4 = factory.getFlyweight("Gooogle");
        fw5 = factory.getFlyweight("Google");
        fw6 = factory.getFlyweight("google");

        fw1.operation();
        fw2.operation();
        fw3.operation();
        fw4.operation();
        fw5.operation();
        fw6.operation();
        System.out.println("Total flyweight size: " + factory.getFlyweightSize());
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/flyweight/flyweight_result.png)

#### 总结

---

&emsp;&emsp;可以看出，虽然有6个享元对象，但实际在容器中只实例化了3个对象，分别是：Google、google和Gooogle，大大减少了占用的内存空间。在JAVA语言中，**String类型就是使用了享元模式**。String对象是final类型，对象一旦创建就不可改变。在JAVA中字符串常量都是存在常量池中的，JAVA会确保一个字符串常量在常量池中只有一个拷贝。String a="abc"，其中"abc"就是一个字符串常量。
