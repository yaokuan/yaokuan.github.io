---
layout:     post
title:      备忘录模式
date:       2018-06-09 15:27:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★☆☆☆，使用频率：★★☆☆☆】

> **备忘录模式**(Memento Pattern)：在不破坏封装的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态，这样可以在以后将对象恢复到原先保存的状态。它是一种对象行为型模式，其别名为Token。




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/memento/memento-uml.png)
在备忘录模式结构图中包含如下几个角色：
1. Originator（原发器）：它是一个普通类，可以创建一个备忘录，并存储它的当前内部状态，也可以使用备忘录来恢复其内部状态，一般将需要保存内部状态的类设计为原发器。
2. Memento（备忘录)：存储原发器的内部状态，根据原发器来决定保存哪些内部状态。备忘录的设计一般可以参考原发器的设计，根据实际需要确定备忘录类中的属性。需要注意的是，除了原发器本身与负责人类之外，备忘录对象不能直接供其他类使用，原发器的设计在不同的编程语言中实现机制会有所不同。
3. Caretaker（负责人）：负责人又称为管理者，它负责保存备忘录，但是不能对备忘录的内容进行操作或检查。在负责人类中可以存储一个或多个备忘录对象，它只负责存储对象，而不能修改对象，也无须知道对象的实现细节。

#### 具体代码

---

Originator角色：
```java
public class Originator {
    private String state;

    public String getState() {
        return state;
    }

    public void setState(String state) {
        this.state = state;
    }

    public Memento saveStateToMemen() {
        return new Memento(state);
    }

    public void restoreStateFromMemto(Memento memento) {
        state = memento.getState();
    }
}
```

Memento角色：

```java
public class Memento {
    private String state;

    public Memento(String state) {
        this.state = state;
    }

    public String getState() {
        return state;
    }
}
```

CareTaker角色：
```java
public class CareTaker {
    private List<Memento> mementoList = new ArrayList<Memento>();

    public void saveState(Memento state) {
        mementoList.add(state);
    }

    public Memento get(int index) {
        return mementoList.get(index);
    }
}
```

测试类Client：
```java
public class Client {
    public static void main(String[] args) {
        Originator originator = new Originator();
        CareTaker careTaker = new CareTaker();
        originator.setState("#1");
        printCurrentState(originator);

        originator.setState("#2");
        printCurrentState(originator);
        careTaker.saveState(originator.saveStateToMemen());

        originator.setState("#3");
        printCurrentState(originator);
        careTaker.saveState(originator.saveStateToMemen());

        originator.setState("#4");
        printCurrentState(originator);
        //restore to savepoint 0
        originator.restoreStateFromMemto(careTaker.get(0));
        System.out.println("Restore to state:" + originator.getState());
        //proceeding
        originator.setState("#5");
        printCurrentState(originator);
        //restore to savepoint 1
        originator.restoreStateFromMemto(careTaker.get(1));
        System.out.println("Restore to state: " + originator.getState());

        originator.setState("#6");
        printCurrentState(originator);
    }

    private static void printCurrentState(Originator originator) {
        System.out.println("Current state: " + originator.getState());
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/memento/memento-result.png)


#### 模式优点
1. 备忘录模式提供了一种**状态恢复的实现机制**，使得用户可以方便地回到一个特定的历史步骤，当新的状态无效或者存在问题时，可以使用暂时存储起来的备忘录将状态复原。
2. 备忘录实现了对信息的封装，一个备忘录对象是一种原发器对象状态的表示，不会被其他代码所改动。备忘录保存了原发器的状态，采用列表、堆栈等集合来存储备忘录对象可以实现多次撤销操作。

#### 模式缺点
**资源消耗过大**，如果需要保存的原发器类的成员变量太多，就不可避免需要占用大量的存储空间，每保存一次对象的状态都需要消耗一定的系统资源。

#### 模式适用场景
1. 保存一个对象在某一个时刻的全部状态或部分状态，这样以后需要时它能够恢复到先前的状态，实现撤销操作。
2. 防止外界对象破坏一个对象历史状态的封装性，避免将对象历史状态的实现细节暴露给外界对象。
