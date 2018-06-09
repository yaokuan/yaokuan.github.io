---
layout:     post
title:      中介者模式
date:       2018-06-09 14:51:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★★☆☆，使用频率：★★☆☆☆】

> **中介者模式**将一个网状的系统结构变成一个以中介者对象为中心的星形结构，在这个星型结构中，使用中介者对象与其他对象的一对多关系来取代原有对象之间的多对多关系。中介者模式定义了一个中介对象来封装一系列对象之间的交互关系。中介者使各个对象之间不需要显式地相互引用，从而使耦合性降低，而且可以独立地改变它们之间的交互行为。

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/mediator/mediator-relations.png)




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/mediator/mediator-uml.png)

#### 具体代码

> 我们来看一个聊天室的场景，每个人都将消息发送给聊天室，然后由聊天室发送个每个人，如果是由每个人相互地转发，产生的就是网状结构，对应于对象间的依赖关系就非常复杂，而中介者模式将这种关系变为了星型结构，每个人都只需要和聊天室产生联系即可。

---

用户Users：
```java
public class User {
    private String name;
    private ChatRoom chatRoom;

    public String getName() {
        return name;
    }

    public User(String name) {
        this.name = name;
    }

    public void enterChatRoom(ChatRoom chatRoom) {
        this.chatRoom = chatRoom;
    }

    public void sendMessage(String message) {
        chatRoom.pushMessage(message, this);
    }
}
```

聊天室ChatRoom：

```java
public class ChatRoom {
    public void pushMessage(String message, User user) {
        System.out.println(String.format("%s [%s]: %s", new Date().toString(), user.getName(), message));
    }
}
```

测试类Client：
```java
public class Client {
    public static void main(String[] args) {
        ChatRoom chatRoom = new ChatRoom();
        User john = new User("John");
        User tom = new User("Tom");
        john.enterChatRoom(chatRoom);
        tom.enterChatRoom(chatRoom);

        john.sendMessage("Hello tom!");
        tom.sendMessage("Hi John, how are u today?");
        john.sendMessage("Never mention about it, I've been fired.");
        tom.sendMessage("Don't be so sad, I believe you can find a better one!");
        john.sendMessage("Thanks, yours words really help!");
        john.sendMessage("I'm gonna have a drink and forget about it, bye.");
        tom.sendMessage("Ok, See u.");
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/mediator/mediator-result.png)


#### 模式优点
1. 中介者模式**简化了对象之间的交互**，它用中介者和同事的一对多交互代替了原来同事之间的多对多交互，一对多关系更容易理解、维护和扩展，将原本难以理解的网状结构转换成相对简单的星型结构。
2. 中介者模式**可将各同事对象解耦**。中介者有利于各同事之间的松耦合，我们可以独立的改变和复用每一个同事和中介者，增加新的中介者和新的同事类都比较方便，更好地符合“开闭原则”。
3. 可以**减少子类生成**，中介者将原本分布于多个对象间的行为集中在一起，改变这些行为只需生成新的中介者子类即可，这使各个同事类可被重用，无须对同事类进行扩展。
 

#### 模式缺点
在具体中介者类中包含了大量同事之间的交互细节，可能会导**致具体中介者类非常复杂**，使得系统难以维护。


#### 模式适用场景
1. 系统中对象之间存在复杂的引用关系，**系统结构混乱且难以理解**。
2. 一个对象由于引用了其他很多对象并且直接和这些对象通信，导致**难以复用该对象**。
3. 想通过一个中间类来封装多个类中的行为，而又不想生成太多的子类。可以通过引入中介者类来实现，在中介者中定义对象交互的公共行为，如果需要改变行为则可以增加新的具体中介者类。
