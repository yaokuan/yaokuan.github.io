---
layout:     post
title:      外观模式
date:       2018-05-27 11:04:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

> **外观（门面）模式（Facade）**,他隐藏了系统的复杂性，并向客户端提供了一个可以访问系统的接口。这种类型的设计模式属于结构性模式。为子系统中的一组接口提供了一个统一的访问接口，这个接口使得子系统更容易被访问或者使用。 



#### 角色
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/facade/facade_structure.png)
简单来说，外观模式就是把一些复杂的流程封装成一个接口供给外部用户更简单的使用。这个模式中，涉及到3个角色：
1. **门面角色**：外观模式的核心。它被客户角色调用，它熟悉子系统的功能。内部根据客户角色的需求预定了几种功能的组合。
2. **子系统角色**：实现了子系统的功能。它对客户角色和Facade时未知的。它内部可以有系统内的相互交互，也可以由供外界调用的接口。
3. **客户角色**：通过调用Facede来完成要实现的功能。

#### 适用范围
1. 为复杂的模块或子系统提供外界访问的模块；
2. 子系统相互独立；
3. 在层析结构中，可以使用外观模式定义系统的每一层的入口。
　　
#### UML图
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/facade/facade_uml.png)
我们来看下具体的实现，从UML图中可以看出，通过客户端Client和Computer的关联关系，客户端只需要操作Computer，而实际由Computer去控制各个子部件的开关机流程，所以Computer即为整个系统的门面，那么废话不多说，**Show me the code!**

#### 具体代码
首先是各个子部件：

```java
public class CPU {
    public void startCPU() {
        System.out.println("CPU is started");
    }

    public void shutdownCPU() {
        System.out.println("CPU is shutdown");
    }
}

public class Disk {
    public void startDisk() {
        System.out.println("Disk is started");
    }

    public void shutdownDisk() {
        System.out.println("Disk is shutdown");
    }
}

public class Memory {
    public void startMemory() {
        System.out.println("Memory is started");
    }

    public void shutdownMemory() {
        System.out.println("Memory is shutdown");
    }
}
```

然后是我们的门店类Computer，聚合了CPU、Disk、Memory，启动Computer时，分别调用各子部件的启动方法，关机时类似。

```java
public class Computer {
    private CPU cpu;
    private Disk disk;
    private Memory memory;

    public Computer() {
        this.cpu = new CPU();
        this.disk = new Disk();
        this.memory = new Memory();
    }

    public void startComputer() {
        System.out.println("Starting computer...");
        cpu.startCPU();
        disk.startDisk();
        memory.startMemory();
        System.out.println("Computer is started");
    }

    public void shutdownComputer() {
        System.out.println("Shutdown computer...");
        cpu.shutdownCPU();
        disk.shutdownDisk();
        memory.shutdownMemory();
        System.out.println("Computer is shutdown");
    }
}
```

最后是客户端类:
```java
public class Client {
    public static void main(String[] args) {
        Computer computer = new Computer();
        //用户需要打开电脑
        computer.startComputer();
        //用户需要关闭电脑
        computer.shutdownComputer();
    }
}
```

看下结果：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/facade/facade_result.png)
