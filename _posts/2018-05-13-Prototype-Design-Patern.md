---
layout:     post
title:      原型模式
date:       2018-05-13 19:23:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

> **原型模式**即用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象。

#### 适用范围
1. 如果创建新对象成本较大，我们可以利用已有的对象进行复制来获得。
2. 如果系统要保存对象的状态，而对象的状态变化很小，或者对象本身占内存不大的时候，也可以使用原型模式配合备忘录模式来应用。相反，如果对象的状态变化很大，或者对象占用的内存很大，那么采用状态模式会比原型模式更好。 
3. 需要避免使用分层次的工厂类来创建分层次的对象，并且类的实例对象只有一个或很少的几个组合状态，通过复制原型对象得到新实例可能比使用构造函数创建一个新实例更加方便。



#### UML图
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/prototypePattern/prototype-uml.png)

原型模式主要包含如下三个角色：
1. **Prototype**：抽象原型类。声明克隆自身的接口。 
2. **ConcretePrototype**：具体原型类。实现克隆的具体操作。 
3. **Client**：客户类。让一个原型克隆自身，从而获得一个新的对象。

#### 具体代码
原型Prototype：

```java
public abstract class Prototype implements Cloneable {
    public Prototype clone() {
        Prototype prototype = null;
        try {
            prototype = (Prototype) super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return prototype;
    }

    public abstract void show();
}

public class ConcretePrototypeA extends Prototype {

    @Override
    public void show() {
        System.out.println("concrete prototype a is created");
    }
}

public class ConcretePrototypeB extends Prototype {

    @Override
    public void show() {
        System.out.println("concrete prototype b is created");
    }
}
```

客户端Client：

```java
public class Client {
    public static void main(String[] args) {
        ConcretePrototypeA cpa = new ConcretePrototypeA();
        ConcretePrototypeB cpb = new ConcretePrototypeB();
        for (int i=1; i<=5; i++) {
            ConcretePrototypeA cpac = (ConcretePrototypeA)cpa.clone();
            cpac.show();
            ConcretePrototypeB cpbc = (ConcretePrototypeB)cpb.clone();
            cpbc.show();
        }

    }
}
```

看下结果：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/prototypePattern/result.png)

#### 需要注意
- 使用原型模式复制对象不会调用类的构造方法。因为对象的复制是通过调用Object类的clone方法来完成的，它直接在内存中复制数据，因此不会调用到类的构造方法。
- 深拷贝与浅拷贝。Object类的clone方法只会拷贝对象中的基本的数据类型（8种基本数据类型byte,char,short,int,long,float,double，boolean），对于数组、容器对象、引用对象等都不会拷贝，这就是浅拷贝。
