---
layout:     post
title:      抽象工厂模式
date:       2018-05-11 23:31:00
categories: 设计模式
tags:       设计模式 工厂模式
---

* 文章目录
{:toc}

> 上一篇我们提到了[工厂方法模式](https://yaokuan.github.io/2018/05/11/Factory-Function-Pattern/)，工厂方法模式面对多种不同的产品簇的时候，产生的类爆炸是我们不能接受的，所以我们接下来介绍一种更灵活的方式



#### 抽象工厂模式
还是同样的形状工厂的例子，我们看一下抽象工厂模式的类图：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/factoryPattern/abstract-factory-pattern.png)

下面是具体的代码：

对于形状类，抽象出接口`Shape`
```java
public interface Shape {
    void draw();
}

public class Rectangle implements Shape {

    @Override
    public void draw() {
        System.out.println("Draw a Rectangle");
    }
}

public class Square implements Shape {

    @Override
    public void draw() {
        System.out.println("Draw a Square");
    }
}

public class Circle implements Shape {

    @Override
    public void draw() {
        System.out.println("Draw a Circle");
    }
}
```

我们将颜色换为形状的大小，抽象出接口`Size`

```java
public interface Size {
    public void show();
}

public class Big implements Size {
    @Override
    public void show() {
        System.out.println("big size");
    }
}

public class Medium implements Size {
    @Override
    public void show() {
        System.out.println("medium size");
    }
}

public class Small implements Size {
    @Override
    public void show() {
        System.out.println("small size");
    }
}
```

最后是工厂类：

```java
public interface Factory {
    Shape produceShape();

    Size produceSize();
}

public class BigCircleFactory implements Factory {
    @Override
    public Shape produceShape() {
        return new Circle();
    }

    @Override
    public Size produceSize() {
        return new Big();
    }
}

public class SmallSquareFactory implements Factory {
    @Override
    public Shape produceShape() {
        return new Square();
    }

    @Override
    public Size produceSize() {
        return new Small();
    }
}
```

测试一下：

```java
public class Test {
    public static void main(String[] args) {
        BigCircleFactory bigCircleFactory = new BigCircleFactory();
        Shape circle = bigCircleFactory.produceShape();
        Size big = bigCircleFactory.produceSize();
        big.show();
        circle.draw();

        SmallSquareFactory smallSquareFactory = new SmallSquareFactory();
        Shape square = smallSquareFactory.produceShape();
        Size small = smallSquareFactory.produceSize();
        small.show();
        square.draw();
    }
}
```

#### 总结：
无论是**简单工厂模式**，**工厂方法模式**，还是**抽象工厂模式**，他们都属于工厂模式，在形式和特点上也是极为相似的，他们的最终目的都是为了解耦。在使用时，我们不必去在意这个模式到底工厂方法模式还是抽象工厂模式，因为他们之间的演变常常是令人琢磨不透的。

经常你会发现，明明使用的工厂方法模式，当新需求来临，稍加修改，加入了一个新方法后，由于类中的产品构成了不同等级结构中的产品族，它就变成抽象工厂模式了；而对于抽象工厂模式，当减少一个方法使的提供的产品不再构成产品族之后，它就演变成了工厂方法模式。

所以，在使用工厂模式时，只需要关心降低耦合度的目的是否达到了就可以了。


