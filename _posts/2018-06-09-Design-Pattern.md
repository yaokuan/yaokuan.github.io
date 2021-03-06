---
layout:     post
title:      浅谈23种设计模式
date:       2018-06-09 17:06:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

# 设计模式系列正式宣告完结！喜大普奔！撒花~~



> **设计模式（Design pattern）** 是一套被反复使用的、多数人知晓的、经过分类编目的、代码设计经验的总结。使用设计模式是为了重用代码、让代码更容易被他人理解、保证代码可靠性。

### 设计模式的类型

我们常说的设计模式是根据四人帮《设计模式》一书中提到的23种设计模式，这些模式可以分为三大类：**创建型模式（Creational Patterns）、结构型模式（Structural Patterns）、行为型模式（Behavioral Patterns）**

#### 创建型模式

模式 | 学习难度 | 使用频率
---|---|---
[单例模式（Singleton Pattern）](https://yaokuan.github.io/2018/05/06/Singleton-Pattern/)|★☆☆☆☆|★★★★☆
[工厂方法模式（Factory Pattern）](https://yaokuan.github.io/2018/05/11/Factory-Function-Pattern/)|★★☆☆☆|★★★★★
[抽象工厂模式（Abstract Factory Pattern）](https://yaokuan.github.io/2018/05/11/Abstract-Factory-Pattern/)|★★★★☆|★★★★★
[原型模式（Prototype Pattern）](https://yaokuan.github.io/2018/05/13/Prototype-Design-Patern/)|★★★☆☆|★★★☆☆
[建造者模式（Builder Pattern）](https://yaokuan.github.io/2018/05/12/Builder-Design-Pattern/)|★★★★☆|★★☆☆☆

#### 结构型模式 

模式 | 学习难度 | 使用频率
---|---|---
[适配器模式（Adapter Pattern）](https://yaokuan.github.io/2018/05/25/Adapter-Design-Pattern/)|★★☆☆☆|★★★★☆
[桥接模式（Bridge Pattern）](https://yaokuan.github.io/2018/05/26/Bridge-Design-Pattern/)|★★★☆☆|★★★☆☆
[组合模式（Composite Pattern）](https://yaokuan.github.io/2018/06/02/Design-Pattern-Composite/)|★★★☆☆|★★★★☆
[装饰器模式（Decorator Pattern）](https://yaokuan.github.io/2018/05/01/decorate-design-pattern/)|★★★☆☆|★★★☆☆
[外观模式（Facade Pattern）](https://yaokuan.github.io/2018/05/27/Facade-Design-Pattern/)|★☆☆☆☆|★★★★★
[享元模式（Flyweight Pattern）](https://yaokuan.github.io/2018/06/02/Flyweight-Design-Pattern/)|★★★★☆|★☆☆☆☆
[代理模式（Proxy Pattern）](https://yaokuan.github.io/2018/06/02/Proxy-Design-Pattern/)|★★★☆☆|★★★★☆

#### 行为型模式

模式 | 学习难度 | 使用频率
---|---|---
[责任链模式（Chain of Responsibility Pattern）](https://yaokuan.github.io/2018/06/02/Chain-Design-Pattern/)|★★★☆☆|★★☆☆☆
[命令模式（Command Pattern）](https://yaokuan.github.io/2018/06/09/Command-Design-Pattern/)|★★★☆☆|★★★★☆
[解释器模式（Interpreter Pattern）](https://yaokuan.github.io/2018/06/09/Intepreter-Design-Pattern/)|★★★★★|★☆☆☆☆
[迭代器模式（Iterator Pattern）](https://yaokuan.github.io/2018/06/09/Iterator-Design-Pattern/)|★★★☆☆|★★★★★
[中介者模式（Mediator Pattern）](https://yaokuan.github.io/2018/06/09/Mediator-Design_Pattern/)|★★★☆☆|★★☆☆☆
[备忘录模式（Memento Pattern）](https://yaokuan.github.io/2018/06/09/Memento-Design-Pattern/)|★★☆☆☆|★★☆☆☆
[观察者模式（Observer Pattern）](https://yaokuan.github.io/2018/06/09/Observer-Design-Pattern/)|★★★☆☆|★★★★★
[状态模式（State Pattern）](https://yaokuan.github.io/2018/06/09/State-Design-Pattern/)|★★★☆☆|★★★☆☆
[策略模式（Strategy Pattern）](https://yaokuan.github.io/2018/06/02/Strategy-Design-Pattern/)|★☆☆☆☆|★★★★☆
[模板方法模式（Template Method Pattern）](https://yaokuan.github.io/2018/06/06/Template-Design-Pattern/)|★★☆☆☆|★★★☆☆
[访问者模式（Visitor Pattern）](https://yaokuan.github.io/2018/06/09/Visitor-Design-Pattern/)|★★★★☆|★☆☆☆☆


**附上模式关系图**

![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/designPattern/design-pattern-relations.jpg)

### 设计模式的六大原则
#### 1、开闭原则（Open Close Principle）

开闭原则的意思是：对扩展开放，对修改关闭。在程序需要进行拓展的时候，不能去修改原有的代码，实现一个热插拔的效果。简言之，是为了使程序的扩展性好，易于维护和升级。想要达到这样的效果，我们需要使用接口和抽象类，后面的具体设计中我们会提到这点。

#### 2、里氏代换原则（Liskov Substitution Principle）

里氏代换原则是面向对象设计的基本原则之一。 里氏代换原则中说，任何基类可以出现的地方，子类一定可以出现。LSP 是继承复用的基石，只有当派生类可以替换掉基类，且软件单位的功能不受到影响时，基类才能真正被复用，而派生类也能够在基类的基础上增加新的行为。里氏代换原则是对开闭原则的补充。实现开闭原则的关键步骤就是抽象化，而基类与子类的继承关系就是抽象化的具体实现，所以里氏代换原则是对实现抽象化的具体步骤的规范。

#### 3、依赖倒转原则（Dependence Inversion Principle）

这个原则是开闭原则的基础，具体内容：针对接口编程，依赖于抽象而不依赖于具体。

#### 4、接口隔离原则（Interface Segregation Principle）

这个原则的意思是：使用多个隔离的接口，比使用单个接口要好。它还有另外一个意思是：降低类之间的耦合度。由此可见，其实设计模式就是从大型软件架构出发、便于升级和维护的软件设计思想，它强调降低依赖，降低耦合。

#### 5、迪米特法则，又称最少知道原则（Demeter Principle）

最少知道原则是指：一个实体应当尽量少地与其他实体之间发生相互作用，使得系统功能模块相对独立。

#### 6、合成复用原则（Composite Reuse Principle）

合成复用原则是指：尽量使用合成/聚合的方式，而不是使用继承。
