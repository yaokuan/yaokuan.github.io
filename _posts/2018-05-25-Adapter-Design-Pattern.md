---
layout:     post
title:      适配器模式
date:       2018-05-25 15:14:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★☆☆☆，使用频率：★★★★☆】

> **适配器模式**将一个类的接口转换成客户希望的另外一个接口。Adapter模式使得原本由于接口不兼容而不能一起工作的那些类可以在一起工作。



#### 适用范围
1. 系统需要使用现有的类，而这些类的接口不符合系统的需要。 
2. 想要建立一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作。 
3. 需要一个统一的输出接口，而输入端的类型不可预知。

#### UML图
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/adapter/adapter_uml.png)
从图中可以看出，有USB和MicroUSB两个接口，而我们分别使用对象适配器AdapterObj和类适配器AdapterClass实现了两种模式，它们都是将MicroUSB接口适配为USB接口，废话不多说，我们来看下代码。

#### 具体代码
首先是需要被适配的MicroUSB接口和需要适配的USB接口：

```java
public interface MicroUSB {
    void isMicroUsb();
}

public interface USB {
    void isUsb();
}
```

然后是USB的实现类UsbPlugin：

```java
public class UsbPlugin implements USB {
    @Override
    public void isUsb() {
        System.out.println("USB口");
    }
}
```

再然后是对象适配器AdapterObj:
```java
public class AdapterObj implements MicroUSB {
    //以聚合的方式实现适配
    private USB usb;

    public AdapterObj(USB usb) {
        this.usb = usb;
    }

    @Override
    public void isMicroUsb() {
        usb.isUsb();
    }
}
```

最后是类适配器AdapterClass:

```java
public class AdapterClass extends UsbPlugin implements MicroUSB {
    //以继承的方式实现适配
    @Override
    public void isMicroUsb() {
        super.isUsb();
    }
}
```


看下结果：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/pattern/adapter/adapter_result.png)
