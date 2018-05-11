---
layout:     post
title:      工厂方法模式
date:       2018-05-11 23:21:00
categories: 设计模式
tags:       设计模式 工厂模式 工厂方法模式
---

* 文章目录
{:toc}

> **工厂模式**主要是为创建对象提供过渡接口，以便将创建对象的具体过程屏蔽隔离起来，达到提高灵活性的目的。 
工厂模式可以分为三类： 

1. 简单工厂模式（Simple Factory） 
2. 工厂方法模式（Factory Method） 
3. 抽象工厂模式（Abstract Factory） 



这三种模式从上到下逐步抽象，并且更具一般性。  
> GOF在《设计模式》一书中将工厂模式分为两类：**工厂方法模式（Factory Method）** 与 **抽象工厂模式（Abstract Factory）**。

将简单工厂模式看为工厂方法模式的一种特例，两者归为一类。 

#### 简单工厂模式
简单工厂模式又称静态工厂方法模式，主要通过建立一个工厂（一个函数或一个类方法）来制造新的对象

我们先来看一下简单工厂模式的类图：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/factoryPattern/simple-factory-pattern.png)

下面是主要代码：

```
public interface Shape {
    void draw();
}

public class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Draw a Circle");
    }
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

public class Factory {
    public static Shape produce(String shape) {
        if (shape.equalsIgnoreCase("square")) {
            return new Square();
        } else if (shape.equalsIgnoreCase("rectangle")) {
            return new Rectangle();
        } else if (shape.equalsIgnoreCase("circle")) {
            return new Circle();
        }
        return null;
    }
}
```

可以看到，通过工厂的静态方法，传递参数并生成具体的对象。但有一个问题：**每次新增一种新的形状，都需要修改工厂类的代码，这显然是违背开闭原则的**。

#### 工厂方法模式
> **工厂方法模式**去掉了简单工厂模式中工厂方法的静态属性，使得它可以被子类继承。这样在简单工厂模式里集中在工厂方法上的压力可以由工厂方法模式里不同的工厂子类来分担。

我们仍然先来看一下类图：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/factoryPattern/factory-function-pattern.png)


Shape、Circle、Rectangle、Square类的代码同上，不再赘述，下面看下工厂具体的代码：
```
public interface Factory {
    Shape produce();
}

public class CircleFactory implements Factory{

    public Circle produce() {
        return new Circle();
    }
}

public class RectangleFactory implements Factory{

    public Rectangle produce() {
        return new Rectangle();
    }
}

public class SquareFactory implements Factory{

    public Square produce() {
        return new Square();
    }
}

public class Test {
    public static void main(String[] args) {
        CircleFactory circleFactory = new CircleFactory();
        Circle circle = circleFactory.produce();
        circle.draw();

        RectangleFactory rectangleFactory = new RectangleFactory();
        Rectangle rectangle = rectangleFactory.produce();
        rectangle.draw();

        SquareFactory squareFactory = new SquareFactory();
        Square square = squareFactory.produce();
        square.draw();
    }
}
```

可以看到工厂方法模式将工厂声明为一个接口，这样具体的实例化就交给了具体的子类，这样新增的产品就可以通过继承工厂接口来生成，一切看来似乎都很完美，但是……

突然有一天，工厂对产品进行了升级，现在不仅有**Circle、Square、Rectangle**，还新增了**Triangle**和**Diamond**，同时每个形状都可以有**Red、Blue、Black、White、Green**五种颜色，那么我们需要**5x5=25**种不同的工厂，我的天!如果继续扩充呢...稍等...我们得另寻出路。下一篇我们将介绍：[抽象工厂模式](https://note.youdao.com/)
