---
layout:     post
title:      迭代器模式
date:       2018-06-09 12:23:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★★☆☆，使用频率：★★★★★】

> **迭代器模式**(Iterator Pattern)：提供一种方法来访问聚合对象，而不用暴露这个对象的内部表示，其别名为游标(Cursor)。迭代器模式是一种对象行为型模式。




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/iterator/iterator-uml.png)

##### 迭代器模式中的角色
1. Iterator（抽象迭代器）：它定义了访问和遍历元素的接口，声明了用于遍历数据元素的方法，例如：用于获取第一个元素的first()方法，用于访问下一个元素的next()方法，用于判断是否还有下一个元素的hasNext()方法，用于获取当前元素的currentItem()方法等，在具体迭代器中将实现这些方法。
2. ConcreteIterator（具体迭代器）：它实现了抽象迭代器接口，完成对聚合对象的遍历，同时在具体迭代器中通过游标来记录在聚合对象中所处的当前位置，在具体实现时，游标通常是一个表示位置的非负整数。
3. Aggregate（抽象聚合类）：它用于存储和管理元素对象，声明一个createIterator()方法用于创建一个迭代器对象，充当抽象迭代器工厂角色。
4. ConcreteAggregate（具体聚合类）：它实现了在抽象聚合类中声明的createIterator()方法，该方法返回一个与该具体聚合类对应的具体迭代器ConcreteIterator实例。

#### 具体代码

---

Iterator角色：
```java
public interface Iterator {
    Object first();

    Object last();

    Object next();

    boolean hasNext();
}
```

Aggregate角色：

```java
public interface Aggregate {
    Iterator iterator();
}
```

ConcreteAggregate角色：

> 这里通过内部类MyIterator来实现MyCollection的双继承，内部类实现了抽象迭代子角色所规定的接口

```java
public class MyCollection implements Aggregate {

    private Object[] objects = null;

    public MyCollection(Object[] objects) {
        this.objects = objects;
    }

    public int size() {
        return objects == null ? 0 : objects.length;
    }

    @Override
    public Iterator iterator() {
        return new MyIterator();
    }

    private class MyIterator implements Iterator{

        private int index;
        private int size;

        public MyIterator() {
            this.index = 0;
            this.size = objects == null ? 0 : objects.length;
        }

        @Override
        public Object first() {
            return objects == null ? null : objects[0];
        }

        @Override
        public Object last() {
            return objects == null ? null : objects[size - 1];
        }

        @Override
        public Object next() {
            if (index >= 0 && index < size) {
                return objects[index++];
            }else {
                return null;
            }
        }

        @Override
        public boolean hasNext() {
            return index < size;
        }
    }
}
```

测试类Client：
```java
public class Client {
    public static void main(String[] args) {
        Object[] objects = new Object[]{"Hello", "world", "I", "am", "Java"};
        MyCollection myCollection = new MyCollection(objects);
        Iterator its = myCollection.iterator();
        while (its.hasNext()) {
            Object next = its.next();
            System.out.print(next);
            System.out.print(" ");
        }
        System.out.println();

        System.out.println(its.first());
        System.out.println(its.last());
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/iterator/iterator-result.png)

#### 总结

---

##### 优点
迭代器模式的主要优点如下：
1. 持以不同的方式遍历一个聚合对象，在同一个聚合对象上可以定义多种遍历方式。在迭代器模式中只需要用一个不同的迭代器来替换原有迭代器即可改变遍历算法，我们也可以自己定义迭代器的子类以支持新的遍历方式。
2. 器简化了聚合类。由于引入了迭代器，在原有的聚合对象中不需要再自行提供数据遍历等方法，这样可以简化聚合类的设计。
3. 代器模式中，由于引入了抽象层，增加新的聚合类和迭代器类都很方便，无须修改原有代码，满足“开闭原则”的要求。

##### 缺点
迭代器模式的主要缺点如下：
1. 迭代器模式将存储数据和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加，这在一定程度上增加了系统的复杂性。
2. 迭代器的设计难度较大，需要充分考虑到系统将来的扩展，例如JDK内置迭代器Iterator就无法实现逆向遍历，如果需要实现逆向遍历，只能通过其子类ListIterator等来实现，而ListIterator迭代器无法用于操作Set类型的聚合对象。在自定义迭代器时，创建一个考虑全面的抽象迭代器并不是件很容易的事情。

##### 适用场景
在以下情况下可以考虑使用迭代器模式：
1. 一个聚合对象的内容而无须暴露它的内部表示。将聚合对象的访问与内部数据的存储分离，使得访问聚合对象时无须了解其内部实现细节。
2. 为一个聚合对象提供多种遍历方式。
3. 历不同的聚合结构提供一个统一的接口，在该接口的实现类中为不同的聚合结构提供不同的遍历方式，而客户端可以一致性地操作该接口。
