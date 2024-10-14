---
layout:     post
title:      Java GC日志简析
date:       2018-08-25 22:24:00
categories: JVM
tags:       Java GC JVM
---

* 文章目录
{:toc}




## 123Java OOM
### Java堆溢出
堆溢出最常见的就是在堆上分配内存而分配对象的内存又GC不掉（为何GC不掉，这部分后面会详细展开），导致新对象无法分配内存而发生OOM。为了演示方便，将Java堆的大小限制为20M且不可扩展，并在发生溢出时打印堆dump以便分析

```java
/**
 * @Author: yaokuan
 * @Date: 2018/8/18 上午9:20
 * -verbose:gc
 * -Xms20m
 * -Xmx20m
 * -Xmn10m
 * -XX:+PrintGCDetails
 * -XX:SurvivorRatio=8
 * -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

    static class OOMObject{

    }

    public static void main(String[] args) {
        ArrayList<OOMObject> list = new ArrayList<>();

        while (true) {
            list.add(new OOMObject());
        }
    }
}
```
由于堆内存才分配20M，所以程序运行一会就会溢出，打印的堆栈如：

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid51191.hprof ...
Heap dump file created [28132916 bytes in 0.182 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:2245)
	at java.util.Arrays.copyOf(Arrays.java:2219)
	at java.util.ArrayList.grow(ArrayList.java:242)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:216)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:208)
	at java.util.ArrayList.add(ArrayList.java:440)
	at com.matthewyao.concurrent.jvm.HeapOOM.main(HeapOOM.java:20)
```
然后我们将dump文件**java_pid51191.hprof**通过MAT打开看一下，占用最多内存的就是OOMObject
![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jvm/oldgen-oom.png)


### Java栈溢出
关于虚拟机栈和本地方法栈，Java虚拟机规范描述了两种异常：
1. 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverFlowError异常
2. 如果虚拟机在扩展栈时如法申请到足够的内存空间，将抛出OutOfMemoryError异常

#### StackOverFlowError

```java
public class StackSOF {
    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        StackSOF stackSOF = new StackSOF();
        try {
            stackSOF.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length: " + stackSOF.stackLength);
            throw e;
        }
    }
}
```
由于stackLeak()的递归调用，导致线程不断入栈，超过虚拟机允许的最大深度所以抛出StackOverFlowError

```
stack length: 10828
Exception in thread "main" java.lang.StackOverflowError
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:11)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
	at com.matthewyao.concurrent.jvm.StackSOF.stackLeak(StackSOF.java:12)
```

#### 理解GC日志

每一种收集器的日志形式都是由它们自身的实现所决定的，换而言之，每个收集器的日志格式都可以不一样。但虚拟机设计者为了方便用户阅读，将各个收集器的日志都维持一定的共性，例如以下两段典型的GC日志：

```
33.125：[GC[DefNew：3324K-＞152K（3712K），0.0025925secs]3324K-＞152K（11904K），0.0031680 secs]

100.667：[FullGC[Tenured：0K-＞210K（10240K），0.0149142secs]4603K-＞210K（19456K），[Perm：2999K-＞2999K（21248K）]，0.0150007 secs][Times：user=0.01 sys=0.00，real=0.02 secs]
```

最前面的数字“33.125：”和“100.667：”代表了GC发生的时间，这个数字的含义是从Java虚拟机启动以来经过的秒数，使用-XX:+PrintGCDateStamps可以打印具体的GC时间点。

GC日志开头的“[GC”和“[Full GC”说明了这次垃圾收集的**停顿类型**，而**不是用来区分新生代GC还是老年代GC**的。如果有“Full”，说明这次GC是发生了Stop-The-World的，例如下面这段新生代收集器ParNew的日志也会出现“[Full GC”（这一般是因为出现了分配担保失败之类的问题，所以才导致STW）。如果是调用System.gc（）方法所触发的收集，那么在这里将显示“[Full GC（System）”。

```
[Full GC 283.736：[ParNew：261599K-＞261599K（261952K），0.0000288 secs]
```

接下来的“[DefNew”、“[Tenured”、“[Perm”表示GC发生的区域，这里显示的区域名称与使用的GC收集是密切相关的，例如上面样例所使用的Serial收集器中的新生代名为“Default New Generation”，所以显示的是“[DefNew”。

如果是ParNew收集器，新生代名称就会变为“[ParNew”，意为“Parallel New Generation”。

如果采用Parallel Scavenge收集器，那它配套的新生代称为“PSYoungGen”，老年代和永久代同理，名称也是由收集器决定的。

后面方括号内部的“**3324K-＞152K（3712K）**”含义是“GC前该内存区域已使用容量-＞GC后该内存区域已使用容量（该内存区域总容量）”。 
而在方括号之外的“3324K-＞152K（11904K）”表示“GC前Java堆已使用容量-＞GC后Java堆已使用容量（Java堆总容量）”。

再往后，“0.0025925 secs”表示该内存区域GC所占用的时间，单位是秒。

有的收集器会给出更具体的时间数据，如“[Times：user=0.01 sys=0.00，real=0.02 secs]”，这里面的user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核态消耗的CPU事件和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。

CPU时间与墙钟时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时，但当系统有多CPU或者多核的话，多线程操作会叠加这些CPU时间，所以读者看到user或sys时间超过real时间是完全正常的。

## 内存分配和回收策略

### 新对象优先在Eden分配

对象都是优先分配在Eden区，为了验证，我们让虚拟机都运行在client模式下，并且都使用Serial / Serial Old GC。

```java
/**
 * @Author: yaokuan
 * @Date: 2018/8/25 下午6:24
 * -client
 * -verbose:gc
 * -XX:+PrintGCDetails
 * -XX:+UseSerialGC
 * -Xms20m
 * -Xmx20m
 * -Xmn10m
 * -XX:SurvivorRatio=8
 */
public class EdenFirst {

    private static final int _MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocate1, allocate2, allocate3, allocate4;
        allocate1 = new byte[2 * _MB];
        allocate2 = new byte[2 * _MB];
        allocate3 = new byte[2 * _MB];
        allocate4 = new byte[4 * _MB];
    }
}
```
从日志上可以看出在分配allocate4时发生了Minor GC，由于Survivor区只有1MB，无法存放2MB的对象，没有对象被回收，且allocate4由于新生代没有足够内存，通过分配担保被直接分配到了老年代

```
[GC[DefNew: 7326K->911K(9216K), 0.0046570 secs] 7326K->5007K(19456K), 0.0046750 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 7416K [0x00000007f9a00000, 0x00000007fa400000, 0x00000007fa400000)
  eden space 8192K,  79% used [0x00000007f9a00000, 0x00000007fa05a388, 0x00000007fa200000)
  from space 1024K,  89% used [0x00000007fa300000, 0x00000007fa3e3dd8, 0x00000007fa400000)
  to   space 1024K,   0% used [0x00000007fa200000, 0x00000007fa200000, 0x00000007fa300000)
 tenured generation   total 10240K, used 4096K [0x00000007fa400000, 0x00000007fae00000, 0x00000007fae00000)
   the space 10240K,  40% used [0x00000007fa400000, 0x00000007fa800020, 0x00000007fa800200, 0x00000007fae00000)
 compacting perm gen  total 21248K, used 2929K [0x00000007fae00000, 0x00000007fc2c0000, 0x0000000800000000)
   the space 21248K,  13% used [0x00000007fae00000, 0x00000007fb0dc630, 0x00000007fb0dc800, 0x00000007fc2c0000)
No shared spaces configured.
```

### 大对象直接进入老年代
在JVM参数中指定-XX:PretenureSizeThreshold=3m，则超过3MB的对象都会直接分配到老年代中

```java
/**
 * @Author: yaokuan
 * @Date: 2018/8/25 下午7:01
 * -client
 * -verbose:gc
 * -XX:+PrintGCDetails
 * -XX:+UseSerialGC
 * -Xms20m
 * -Xmx20m
 * -Xmn10m
 * -XX:SurvivorRatio=8
 * -XX:PretenureSizeThreshold=3m
 */
public class BigObj {

    private static final int _MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocation = new byte[4 * _MB];
    }
}
```
可以看到tenured generation内存占用刚好4MB

```
Heap
 def new generation   total 9216K, used 3394K [0x00000007f9a00000, 0x00000007fa400000, 0x00000007fa400000)
  eden space 8192K,  41% used [0x00000007f9a00000, 0x00000007f9d50918, 0x00000007fa200000)
  from space 1024K,   0% used [0x00000007fa200000, 0x00000007fa200000, 0x00000007fa300000)
  to   space 1024K,   0% used [0x00000007fa300000, 0x00000007fa300000, 0x00000007fa400000)
 tenured generation   total 10240K, used 4096K [0x00000007fa400000, 0x00000007fae00000, 0x00000007fae00000)
   the space 10240K,  40% used [0x00000007fa400000, 0x00000007fa800010, 0x00000007fa800200, 0x00000007fae00000)
 compacting perm gen  total 21248K, used 2923K [0x00000007fae00000, 0x00000007fc2c0000, 0x0000000800000000)
   the space 21248K,  13% used [0x00000007fae00000, 0x00000007fb0daca0, 0x00000007fb0dae00, 0x00000007fc2c0000)
No shared spaces configured.
```

### 新生代对象晋升至老年代
虚拟机给每个对象定义了一个对象年龄，对象每经历一次Minor GC，对象年龄都会加1，当对象年龄达到一定的程度（默认是15），虚拟机就会认为该对象会长期存活，理应晋升至老年代，对象晋升至老年代的年龄阈值可以通过JVM参数-XX:MaxTenuringThreshold指定，我们来看下下面这段代码分别在晋升年龄为1和15下的表现。


```java
/**
 * @Author: yaokuan
 * @Date: 2018/8/25 下午7:14
 * -client
 * -verbose:gc
 * -XX:+PrintGCDetails
 * -XX:+UseSerialGC
 * -Xms20m
 * -Xmx20m
 * -Xmn10m
 * -XX:SurvivorRatio=8
 * -XX:MaxTenuringThreshold=1 or 15
 * -XX:+PrintTenuringDistribution
 */
public class ObjectAge {

    private static final int _MB = 1024 * 1024;

    public static void main(String[] args) {
        byte[] allocation1, allocation2, allocation3;
        allocation1 = new byte[_MB / 4];
        allocation2 = new byte[4 * _MB];
        allocation3 = new byte[4 * _MB];
        allocation3 = null;
        allocation3 = new byte[4 * _MB];
    }
}
```
当MaxTenuringThreshold=1时，第一次Minor GC后对象的年龄加1，第二次Minor GC后就直接到老年代，所以新生代的内存都被清空。
当MaxTenuringThreshold=15时，前两次Minor GC都不会使allocation1占用的256KB内存对象进入老年代，所以新生代仍占用了内存。

## 附上Java GC CheatSheet
![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jvm/Java%207%20-%20GC%20cheatsheet.png)

