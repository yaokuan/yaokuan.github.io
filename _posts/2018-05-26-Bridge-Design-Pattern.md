---
layout:     post
title:      桥接模式
date:       2018-05-26 18:19:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

> **桥接模式**即将抽象部分与它的实现部分分离开来，使他们都可以独立变化。桥接模式将继承关系转化成关联关系，它降低了类与类之间的耦合度，减少了系统中类的数量，也减少了代码量。



#### 适用范围
1. 如果一个系统需要在构件的抽象化角色和具体化角色之间增加更多的灵活性，避免在两个层次之间建立静态的继承联系，通过桥接模式可以使它们在抽象层建立一个关联关系。
2. 对于那些不希望使用继承或因为多层次继承导致系统类的个数急剧增加的系统，桥接模式尤为适用。
3. 一个类存在两个独立变化的维度，且这两个维度都需要进行扩展。

#### UML图
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/bridge/bridge_uml.png)
我们来看下具体的实现，从UML图中可以看出，通过类的关联把Color和Shape连接在了一起，每一个实现了Shape接口的对象中都有一个Color的实现，有效地降低了类与类之间的耦合，同时新增一种颜色或者新增一种形状都不会现有结构造成影响，那么废话不多说，我们来看下具体的实现代码。

#### 具体代码
首先颜色Color接口和其子类：

```java
public interface Color {
    void paint(String shape);
}

public class Blue implements Color {
    @Override
    public void paint(String shape) {
        System.out.println("蓝色的" + shape);
    }
}

public class Red implements Color {
    @Override
    public void paint(String shape) {
        System.out.println("红色的" + shape);
    }
}

public class White implements Color {
    @Override
    public void paint(String shape) {
        System.out.println("白色的" + shape);
    }
}
```

其次是形状Shape抽象类和其子类，这里把Shape定义为一个抽象类是由于Shape需要持有一个Color的引用：

```java
public abstract class Shape {
    Color color;

    public void setColor(Color color) {
        this.color = color;
    }

    public abstract void draw();
}

public class Circle extends Shape {
    @Override
    public void draw() {
        color.paint("圆形");
    }
}

public class Rectangle extends Shape {
    @Override
    public void draw() {
        color.paint("长方形");
    }
}

public class Square extends Shape {
    @Override
    public void draw() {
        color.paint("正方形");
    }
}
```

最后是测试类:
```java
public class Test {
    public static void main(String[] args) {
        Color white = new White();
        Color red = new Red();
        Shape rectangle = new Rectangle();
        rectangle.setColor(white);
        rectangle.draw();
        rectangle.setColor(red);
        rectangle.draw();

        Shape circle = new Circle();
        circle.setColor(white);
        circle.draw();
        circle.setColor(red);
        circle.draw();
    }
}
```

看下结果：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/bridge/bridge_result.png)
