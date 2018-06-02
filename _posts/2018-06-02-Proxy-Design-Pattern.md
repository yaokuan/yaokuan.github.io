---
layout:     post
title:      代理模式
date:       2018-06-02 15:42:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★★☆☆，使用频率：★★★★☆】

> **代理模式**给目标对象提供一个代理对象，并由代理对象控制对目标对象的引用




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/proxy/proxy_uml.png)
代理模式中的角色：
1. Subject：公共接口，声明了真实对象和代理对象的公共接口。 
2. RealSubject：真实对象，代理对象所代理的真实对象，最终引用的对象。 
3. Proxy：代理对象，包含对真实对象的引用从而操作真实主题对象，对真实对象进行了封装。

结合UML图，这三个角色不是显而易见的吗?

#### 具体代码

---
&emsp;&emsp;有一天我想买一辆宝马，但我不知道该去哪里买，于是我找到了4s店的小王，委托小王帮我买一辆宝马5系。小王在接受我的委托后，在宝马官网订购了一辆车，等车到了4s店后，他又亲自送到了我家楼下，我很开心。过了一段时间，我又想买一辆奔驰，所以这次我二话不说又找到了小王，而小王也同样帮我预定了一辆奔驰GLA后送到了我家楼下。我很开心，非常感谢小王！**【本故事纯属虚构】**

瞎扯至此，**Show me the code!**

---


Subject角色：

```java
public interface BuySomething {
    void buyBMW();
    void buyBenz();
}
```

RealSubject角色：

```java
public class People implements BuySomething {
    @Override
    public void buyBMW() {
        System.out.println("I want to buy a BMW, but I don't know how, so I find the proxy");
    }

    @Override
    public void buyBenz() {
        System.out.println("I want to buy a Benz, I find the proxy again");
    }
}
```

Proxy角色:
```java
public class Proxy implements BuySomething {

    private People people;

    public Proxy(People people) {
        this.people = people;
    }

    @Override
    public void buyBMW() {
        people.buyBMW();
        doProxy("BMW");
    }

    @Override
    public void buyBenz() {
        people.buyBenz();
        doProxy("Benz");
    }

    private void doProxy(String carName) {
        System.out.println("Proxy booking a "+ carName +"...");
        System.out.println("Proxy Receive the car");
        System.out.println("Proxy send the car to my place...");
        System.out.println("I received my car!");
    }
}
```

测试类Client:

```java
public class Client {
    public static void main(String[] args) {
        People me = new People();
        Proxy proxy = new Proxy(me);
        System.out.println("One day");
        proxy.buyBMW();
        System.out.println();
        System.out.println("After a few days");
        proxy.buyBenz();
    }
}
```

#### 结果

---


![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/proxy/proxy_result.png)
