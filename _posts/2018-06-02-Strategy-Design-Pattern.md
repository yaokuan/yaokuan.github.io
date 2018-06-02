---
layout:     post
title:      策略模式
date:       2018-06-02 18:02:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★☆☆☆☆，使用频率：★★★★☆】

>把一个类中经常改变或者将来可能改变的部分提取出来，作为一个接口，然后在类中包含这个对象的实例，这样类的实例在运行时就可以随意调用实现了这个接口的类的行为。比如定义一系列的算法,把每一个算法封装起来,并且使它们可相互替换，使得算法可独立于使用它的客户而变化。这就是**策略模式**。




#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/strategy/strategy_uml.png)

##### 策略模式中的角色
1. 环境类(Context):用一个ConcreteStrategy对象来配置。维护一个对Strategy对象的引用。可定义一个接口来让Strategy访问它的数据。
2. 抽象策略类(Strategy):定义所有支持的算法的公共接口。 Context使用这个接口来调用某ConcreteStrategy定义的算法。
3. 具体策略类(ConcreteStrategy):以Strategy接口实现某具体算法。

#### 具体代码

---
> 刘备要到江东娶老婆了，走之前诸葛亮给赵云三个锦囊妙计，说是按天机拆开能解决棘手问题。场景中出现三个要素：三个妙计（具体策略类）、一个锦囊（环境类）、赵云（调用者）。

Strategy角色：

```java
public interface IStrategy {
    void operate();
}
```

ConcreteStrategy角色：
```java
public class BackDoor implements IStrategy {
    @Override
    public void operate() {
        System.out.println("找乔国老帮忙，让吴国太给孙权施加压力，使孙权不能杀刘备");
    }
}

public class GivenGreenLight implements IStrategy {
    @Override
    public void operate() {
        System.out.println("求吴国太开个绿灯，放行");
    }
}

public class BlockEnemy implements IStrategy {
    @Override
    public void operate() {
        System.out.println("孙夫人断后，挡住追兵");
    }
}
```

Context角色：

```java
public class Context {
    private IStrategy strategy;

    public void setStrategy(IStrategy strategy) {
        this.strategy = strategy;
    }

    public void operate() {
        strategy.operate();
    }
}
```

使用锦囊的人:

```java
public class Zhaoyun {
    public static void main(String[] args) {
        Context liubeiMarry = new Context();

        System.out.println("----------刚到吴国使用第一个锦囊---------------");
        liubeiMarry.setStrategy(new BackDoor());
        liubeiMarry.operate();
        System.out.println();

        System.out.println("----------刘备乐不思蜀使用第二个锦囊---------------");
        liubeiMarry.setStrategy(new GivenGreenLight());
        liubeiMarry.operate();
        System.out.println();


        System.out.println("----------孙权的追兵来了，使用第三个锦囊---------------");
        liubeiMarry.setStrategy(new BlockEnemy());
        liubeiMarry.operate();
        System.out.println();
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/strategy/strategy_result.png)

三招下来，搞得的周郎是“赔了夫人又折兵”。
