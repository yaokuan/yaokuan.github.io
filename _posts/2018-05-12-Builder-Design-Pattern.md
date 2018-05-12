---
layout:     post
title:      建造者模式
date:       2018-05-12 15:48:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

> **建造者模式**主要通过将一个复杂的对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

#### 适用范围
1. 当创建复杂对象的算法应该独立于该对象的组成部分以及它们的装配方式时。
2. 当构造过程必须允许被构造的对象有不同表示时。



#### UML图
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/builderPattern/builder-uml.png)

在建造者模式中，有以下几个角色：
1. **Builder**：为创建一个产品对象的各个部件指定抽象接口。
2. **ConcreteBuilder**：实现Builder的接口以构造和装配该产品的各个部件，并返回具体产品的一个实例，本例中如：FruitSaladBuilder。
3. **Director**：构造一个使用Builder接口的对象，指导构建过程。
4. **Product**：表示被构造的复杂对象。ConcreteBuilder创建该产品的内部表示并定义它的装配过程，包含定义组成部件的类，包括将这些部件装配成最终产品的接口，本例中产品如：FruitSalad。

#### 具体代码
首先是构建者Builer：

```java
public interface Builder {
    void addVegetables();
    void addAmaranth();
    void addSauce();
    Salad buildSalad();
}

public class CaesarSaladBuilder implements Builder {
    private Salad salad;

    public CaesarSaladBuilder() {
        this.salad = new CaesarSalad();
    }

    @Override
    public void addVegetables() {
        salad.setVegetables("tomato");//西红柿
    }

    @Override
    public void addAmaranth() {
        salad.setAmaranth("roast chicken");//烤鸡肉
    }

    @Override
    public void addSauce() {
        salad.setSauce("yogurt");//酸奶
    }

    @Override
    public Salad buildSalad() {
        addAmaranth();
        addVegetables();
        addSauce();
        return salad;
    }
}

public class FruitSaladBuilder implements Builder {

    private Salad salad;

    public FruitSaladBuilder() {
        this.salad = new FruitSalad();
    }

    @Override
    public void addVegetables() {
        salad.setVegetables("broccoli");//西兰花
    }

    @Override
    public void addAmaranth() {
        salad.setAmaranth("sole file");//龙利鱼
    }

    @Override
    public void addSauce() {
        salad.setSauce("sesame soy");//芝麻酱
    }

    @Override
    public Salad buildSalad() {
        addAmaranth();
        addVegetables();
        addSauce();
        return salad;
    }
}
```

然后是具体产品：

```java
public class Salad {
    private String vegetables;
    private String amaranth;
    private String sauce;

    public String getVegetables() {
        return vegetables;
    }

    public void setVegetables(String vegetables) {
        this.vegetables = vegetables;
    }

    public String getAmaranth() {
        return amaranth;
    }

    public void setAmaranth(String amaranth) {
        this.amaranth = amaranth;
    }

    public String getSauce() {
        return sauce;
    }

    public void setSauce(String sauce) {
        this.sauce = sauce;
    }
}

public class CaesarSalad extends Salad {

    public CaesarSalad() {
        System.out.println("Begin to build caesar salad");
    }
}

public class FruitSalad extends Salad {

    public FruitSalad() {
        System.out.println("Begin to build fruit salad");
    }
}
```

接着是Director：

```java
public class Director {

    private Builder builder;

    public Director(Builder builder) {
        this.builder = builder;
    }

    public Salad getSalad() {
        return builder.buildSalad();
    }
}
```

我们来测试一下：

```java
public class Test {
    /**
     * 将对象表示和对象创建分离，适用于对象组成类似，但又各自有不同的组装过程的场景
     */
    public static void main(String[] args) {
        Builder builder = new FruitSaladBuilder();
        Director director = new Director(builder);
        Salad fruitSalad = director.getSalad();
        System.out.println(fruitSalad);

        builder = new CaesarSaladBuilder();
        director = new Director(builder);
        Salad caesarSalad = director.getSalad();
        System.out.println(caesarSalad);
    }
}
```

最后的结果是：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/5/builderPattern/result.png)

#### 总结
建造者模式在使用过程中可以演化出多种形式：
- 如果具体的被建造对象只有一个的话，可以省略抽象的Builder和Director，让ConcreteBuilder自己扮演指导者和建造者双重角色，甚至ConcreteBuilder也可以放到Product里面实现。
- 在《Effective Java》书中第二条，就提到“遇到多个构造器参数时要考虑用构建器”，其实这里的构建器就属于建造者模式，只是里面把四个角色都放到具体产品里面了。

