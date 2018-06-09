---
layout:     post
title:      命令模式
date:       2018-06-09 11:37:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★★☆☆，使用频率：★★★★☆】

> **命令模式**将来自客户端的请求传入一个对象，从而使你可用不同的请求对客户进行参数化。用于“行为请求者”与“行为实现者”解耦，可实现二者之间的松耦合，以便适应变化。分离变化与不变的因素。




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/command/command-uml.png)

##### 命令模式中的角色
1. Command：定义命令的接口，声明执行的方法。
2. ConcreteCommand：命令接口实现对象，是“虚”的实现；通常会持有接收者，并调用接收者的功能来完成命令要执行的操作。
3. Receiver：接收者，真正执行命令的对象。任何类都可能成为一个接收者，只要它能够实现命令要求实现的相应功能。
4. Invoker：要求命令对象执行请求，通常会持有命令对象，可以持有很多的命令对象。这个是客户端真正触发命令并要求命令执行相应操作的地方，也就是说相当于使用命令对象的入口。
5. Client：创建具体的命令对象，并且设置命令对象的接收者。注意这个不是我们常规意义上的客户端，而是在组装命令对象和接收者，或许，把这个Client称为装配者会更好理解，因为真正使用命令的客户端是从Invoker来触发执行。

#### 具体代码

---
> 由遥控器按钮发出命令控制电视机，用户只关注遥控器，不关心电视机是如何去开机、关机和换台的。

Receiver角色（电视机）：
```java
public class TV {
    private int channel = 0;

    public void trunOn() {
        System.out.println("Turn on the tv.");
    }

    public void turnOff() {
        System.out.println("Turn off the tv.");
    }

    public void changeChannel(int channel) {
        this.channel = channel;
        System.out.println("Now the channel is: " + channel);
    }
}
```

Command角色：

```java
public interface Command {
    void execute();
}

public class CommandOn implements Command {

    private TV tv;

    public CommandOn(TV tv) {
        this.tv = tv;
    }

    @Override
    public void execute() {
        tv.trunOn();
    }
}

public class CommandOff implements Command {

    private TV tv;

    public CommandOff(TV tv) {
        this.tv = tv;
    }

    @Override
    public void execute() {
        tv.turnOff();
    }
}

public class CommandChangeChannel implements Command {

    private TV tv;
    private int channel;

    public CommandChangeChannel(TV tv) {
        this.tv = tv;
    }

    public void setChannel(int channel) {
        this.channel = channel;
    }

    @Override
    public void execute() {
        tv.changeChannel(channel);
    }
}
```

Invoker角色（遥控器）：
```java
public class Control {
    private Command commandOn, commandOff;
    private CommandChangeChannel commandChangeChannel;

    public Control(Command commandOn, Command commandOff, CommandChangeChannel commandChangeChannel) {
        this.commandOn = commandOn;
        this.commandOff = commandOff;
        this.commandChangeChannel = commandChangeChannel;
    }

    public void turnOnTv() {
        commandOn.execute();
    }

    public void turnOffTv() {
        commandOff.execute();
    }

    public void changeChannel(int channel) {
        commandChangeChannel.setChannel(channel);
        commandChangeChannel.execute();
    }
}
```

测试类Client：
```java
public class Client {
    public static void main(String[] args) {
        TV myTv = new TV();

        Command commandOn = new CommandOn(myTv);
        Command commandOff = new CommandOff(myTv);
        CommandChangeChannel commandChangeChannel = new CommandChangeChannel(myTv);

        Control control = new Control(commandOn, commandOff, commandChangeChannel);
        control.turnOnTv();
        control.changeChannel(5);
        control.changeChannel(34);
        control.turnOffTv();
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/command/command-result.png)

#### 总结

---

##### 优点
1. **降低系统的耦合度**。由于请求者与接收者之间不存在直接引用，因此请求者与接收者之间实现完全解耦，相同的请求者可以对应不同的接收者，同样，相同的接收者也可以供不同的请求者使用，两者之间具有良好的独立性。
2. **新的命令可以很容易地加入到系统中**。由于增加新的具体命令类不会影响到其他类，因此增加新的具体命令类很容易，无须修改原有系统源代码，甚至客户类代码，满足“开闭原则”的要求。
3. 可以比较容易地设计一个**命令队列**或宏命令（组合命令）。

##### 缺点
使用命令模式可能会导致某些系统有过多的具体命令类。

##### 适用场景
1. 系统需要将请求调用者和请求接收者解耦，使得调用者和接收者不直接交互。请求调用者无须知道接收者的存在，也无须知道接收者是谁，接收者也无须关心何时被调用。
2.  系统需要在不同的时间指定请求、将请求排队和执行请求。一个命令对象和请求的初始调用者可以有不同的生命期，换言之，最初的请求发出者可能已经不在了，而命令对象本身仍然是活动的，可以通过该命令对象去调用请求接收者，而无须关心请求调用者的存在性，可以通过请求日志文件等机制来具体实现。
3. 系统需要将一组操作组合在一起形成宏命令。
