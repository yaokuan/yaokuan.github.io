---
layout:     post
title:      模板方法模式
date:       2018-06-06 20:45:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★☆☆☆，使用频率：★★★☆☆】

> **模板方法模式**是类的行为模式。准备一个抽象类，将部分逻辑以具体方法以及具体构造函数的形式实现，然后声明一些抽象方法来迫使子类实现剩余的逻辑。不同的子类可以以不同的方式实现这些抽象方法，从而对剩余的逻辑有不同的实现。这就是模板方法模式的用意。
比如定义一个操作中的算法的骨架，将步骤延迟到子类中。模板方法使得子类能够不去改变一个算法的结构即可重定义算法的某些特定步骤。




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/template/template-uml.png)

##### 模板方法模式中的角色
1. 抽象类（AbstractClass）：实现了模板方法，定义了算法的骨架。
2. 具体类（ConcreteClass)：实现抽象类中的抽象方法，已完成完整的算法。

#### 具体代码

---
> 一个具体的例子，每个人醒来后都是先起床，吃早餐，然后去工作（上学），这样就会有一个模板方法，包含抽象类和具体类。

AbstractPerson抽象类角色：

```java
public abstract class AbstractPerson {
    //final修饰防止被修改
    public final void spendAfternoon() {
        getUp();
        eatBreakfast();
        goToWork();
        System.out.println();
    }

    public abstract void getUp();

    public abstract void eatBreakfast();

    public abstract void goToWork();
}
```

具体对象：
```java
public class Children extends AbstractPerson {
    @Override
    public void getUp() {
        System.out.println("被妈妈叫起来、换衣服");
    }

    @Override
    public void eatBreakfast() {
        System.out.println("吃妈妈做的爱心早餐");
    }

    @Override
    public void goToWork() {
        System.out.println("坐校车去上学");
    }
}

public class Teacher extends AbstractPerson {
    @Override
    public void getUp() {
        System.out.println("洗漱、换衣服");
    }

    @Override
    public void eatBreakfast() {
        System.out.println("吃了：面包+鸡蛋");
    }

    @Override
    public void goToWork() {
        System.out.println("开车去学校");
    }
}

public class Programmer extends AbstractPerson {
    @Override
    public void getUp() {
        System.out.println("玩手机、刷牙");
    }

    @Override
    public void eatBreakfast() {
        System.out.println("随便吃一点");
    }

    @Override
    public void goToWork() {
        System.out.println("肯定是挤地铁去上班啊");
    }
}
```

测试类Client：
```java
public class Client {
    public static void main(String[] args) {
        Teacher teacher = new Teacher();
        System.out.println("老师如果度过上午：");
        teacher.spendAfternoon();

        Programmer programmer = new Programmer();
        System.out.println("程序员如果度过上午：");
        programmer.spendAfternoon();

        Children children = new Children();
        System.out.println("小孩如果度过上午：");
        children.spendAfternoon();
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/template/template-result.png)

#### 总结

---

可以看出，模板方法模式在于抽象类中的模板方法，定义了一系列的方法流程，去约定子类的实现步骤，像是一个模板一样，子类依照这个模板去实现某一个具体的目标

##### 优点
模板方法模式通过把不变的行为搬移到超类，去除了子类中的重复代码。
子类实现算法的某些细节，有助于算法的扩展。
通过一个父类调用子类实现的操作，通过子类扩展增加新的行为，符合“开放-封闭原则”。

##### 缺点
每个不同的实现都需要定义一个子类，这会导致类的个数的增加，设计更加抽象。

##### 适用场景
在某些类的算法中，用了相同的方法，造成代码的重复。
控制子类扩展，子类必须遵守算法规则。
