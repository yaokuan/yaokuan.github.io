---
layout:     post
title:      Java内存模型和volatile关键字
date:       2018-08-11 18:24:00
categories: 多线程
tags:       Java 多线程 JVM
---

* 文章目录
{:toc}




## 内存模型简介
在介绍Java的内存模型之前，需要先介绍下物理计算机中的并发问题。我们要知道绝大多数的运算任务都不可能只靠处理器的“计算”就可以完成，处理器至少还要完成内存交互：如读取运算数据、存储运算结果等I/O操作，由于计算机的存储设备和处理器的运算速度差了几个数量级，所以现代计算机系统普遍都在CPU与主存之间加入一层寄存器作为高速缓存（Cache）。虽然缓存能解决内存和CPU的速度矛盾，却引入了一个新的问题：缓存一致性。所以多处理器如何去更新同一主存就涉及到很多同步协议。

那么在Java虚拟机规范中定义了一种Java内存模型来屏蔽不同硬件和操作系统的差异，使得Java在各种平台下都能达到一致的内存访问效果。Java内存模型规定了所有变量都存储在主内存（虚拟机内存的一部分）中，每个线程都有自己的工作内存，线程的工作内存中保存了被该线程用到的变量的主内存副本拷贝，线程对变量的所有操作都是在工作内存中进行的，而不能直接读写主内存中的变量。不同线程间也不能访问对方的工作内存。

![JMM](http://oc26wuqdw.bkt.clouddn.com/blog/threadsync/jmm.jpgjmm.jpg)

这里的主内存、工作内存和Java内存区域中的Java堆、栈、方法区并不是同一个层次的内存划分，两者基本没关系，不过可以理解为主内存主要对应于Java堆中的对象实例数据部分，而工作内存则对应于虚拟机栈中的部分区域。

### 内存间的交互操作
- **lock（锁定）**：作用于主内存的变量，把一个变量标识为一条线程独占的状态。
- **unclock（解锁）**：作用于主内存的变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- **read（读取）**：作用于主内存的变量，把一个变量的值从主内存传输到线程的工作内存，以便随后的load动作使用。
- **load（载入）**：作用于工作内存的变量，把read操作从主内存中得到的变量值放入工作内存的变量副本中。
- **use（使用）**：作用于工作内存的变量，把工作内存中一个变量的值传递给执行引擎。
- **assign（赋值）**：作用于工作内存的变量，把执行引擎接收到的值赋给工作内存的变量。
- **store（存储）**：作用于工作内存的变量，把工作内存中一个变量的值传送给主内存中，以便随后的write操作使用。
- **write（写入）**：作用于主内存的变量，把store操作从工作内存中得到的变量的值放入主内存的变量中。

以上的8种操作还需要满足以下规则：
- 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存中读取了但工作内存不接受，或者工作内存发起了回写但主内存不接受的情况出现。
- 不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步到主内存。
- 不允许一个线程无原因地（没有发生任何assign操作）把数据从线程的工作内存同步到主内存。
- 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说，就是对一个变量use、store操作之前，必须先执行过了assign和load操作。
- 一个变量在同一时刻只允许一个线程对其进行lock操作，但lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unclock操作，变量才会被解锁。
- 如果对一个变量执行lock操作，那么会清空工作内存次变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
- 如果对一个变量执行lock操作，那么会清空工作内存次变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
- 对一个变量执行unclock之前，必须先把此变量同步回主内存中（执行store、write操作）。

## volatile关键字
当我们把一个变量声明为volatile之后，这个变量就变得十分特殊，我们可以把对单个volatile变量的读写看做是单个监视器锁对这个变量的读写操作做了同步，为了理解这个过程，我们先来看一段代码

```java
class VolatileFeaturesExample {
    volatile long vl = 0L;  //使用volatile声明64位的long型变量

    public void set(long l) {
        vl = l;   //单个volatile变量的写
    }

    public void getAndIncrement () {
        vl++;    //复合（多个）volatile变量的读/写
    }


    public long get() {
        return vl;   //单个volatile变量的读
    }
}
```
假设上面这段程序是处在多线程环境中运行，那么对上面三个方法的访问和下面这段代码有相同的内存语义

```java
class VolatileFeaturesExample {
    long vl = 0L;               // 64位的long型普通变量

    public synchronized void set(long l) {     //对单个的普通 变量的写用同一个监视器同步
        vl = l;
    }

    public void getAndIncrement () { //普通方法调用
        long temp = get();           //调用已同步的读方法
        temp += 1L;                  //普通写操作
        set(temp);                   //调用已同步的写方法
    }
    public synchronized long get() { 
    //对单个的普通变量的读用同一个监视器同步
        return vl;
    }
}
```
对一个volatile变量的读写和使用同一个监视器锁对一个普通变量进行同步，具有相同的执行效果。监视器锁的**happens-before规则**确保了释放监视器和获得监视器的两个线程之间的内存可见性，而volatile修饰的变量也具有相同的语义，即使是64位的long型和double型变量，只要它是volatile变量，对该变量的读写就将具有原子性。如果是多个volatile操作或类似于volatile++这种复合操作，这些操作整体上不具有原子性。简而言之，volatile变量自身具有下列特性：
- **可见性**：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
- **原子性**：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。

### volatile特性一：内存可见性
变量的可见性是说当一个线程修改了这个变量的值，其他线程可以立刻获取到最新的值。而普通变量是需要通过高速缓存回写到主内存，并不是立即可见。
通过前文我们知道CPU内核都会有自己的高速缓存区，当内核运行的线程执行一段代码时，首先将这段代码的指令集加载到高速缓存，如果非volatil变量当CPU执行修改了此变量之后，会将修改后的值回写到高速缓存，然后再刷新到内存中。如果在刷新回内存之前，由于是共享变量，那么CORE2中的线程执行的代码也用到了这个变量，这是变量的值依然是旧的（仍不是立即可见）。volatile关键字就会解决这个问题的，被volatile关键字修饰的共享变量在转换成汇编语言时，会加上一个以lock为前缀的指令，当CPU发现这个指令时，立即做两件事：
1. 将当前内核高速缓存行的数据立刻回写到内存；
2. 使在其他内核里缓存了该内存地址的数据无效（缓存一致性协议）。

### volatile非线程安全
虽然volatile修饰的变量具有内存可见性，但是volatile修饰的变量并不是线程安全的，除了读写等操作是原子性外，像自增这种非原子操作就可能导致volatile变量非线程安全，看一段代码：

```java
public class TestVolatile {

    public static volatile int race = 0;

    public static void increse(){
        race++;
    }

    private static final int THREAD_COUNT = 30;

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        Thread[] threads = new Thread[THREAD_COUNT];
        for (int i=0; i<THREAD_COUNT; i++){
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i=0; i<10000; i++){
                        increse();
                    }
                }
            });
            threads[i].start();
        }

        //每次当主线程执行到这里的时候，主线程让出CPU资源，让所有线程（包括主线程）竞争CPU资源
        while (Thread.activeCount() > 2){
            Thread.yield();
        }

        //正常情况下每个线程都加了10000次，结果理应为300000，但由于自增非原子性，结果总会比300000小
        System.out.println(race);
        System.out.println("cost: " + (System.currentTimeMillis() - start ) + " mills");
    }
}
```
虽然race++只有一行代码，但实际执行这行代码在内存语义上等同于

```java
int race = get();
race += 1;
set(race);
```
我们看下执行结果：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jmm-volatile/volatile-result.png)
get()操作将race的值从内存load到工作内存副本中时，volatile关键字保证了race变量在此时是正确的，但当前线程在执行race += 1和set(race)时，其他线程很可能已经把race的值加大了，所以当前线程在工作内存副本中的值就变成了过期的数据，所以set(race)就可能将较小的race值从工作内存写回到主内存中了。

### volatile特性二：禁止指令重排序

在说指令重排序前，我们需要先说一下Java里的先行发生原则
> 先行发生是Java内存模型中定义的两项操作之间的偏序关系，如果说操作A先行发生于操作B，其实就是说在发生操作B之前，操作A产生的影响能被B观察到，“影响”包括修改了内存中共享变量的值，发送了消息、调用了方法等。

1. **程序次序规则**：同一个线程内，按照代码出现的顺序，前面的代码先行于后面的代码，准确的说是控制流顺序，因为要考虑到分支和循环结构。

2. **管程锁定规则**：一个unlock操作先行发生于后面（时间上）对同一个锁的lock操作。

3. **volatile变量规则**：对一个volatile变量的写操作先行发生于后面（时间上）对这个变量的读操作。

4. **线程启动规则**：Thread的start( )方法先行发生于这个线程的每一个操作。

5. **线程终止规则**：线程的所有操作都先行于此线程的终止检测。可以通过Thread.join( )方法结束、Thread.isAlive( )的返回值等手段检测线程的终止。 

6. **线程中断规则**：对线程interrupt( )方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupt( )方法检测线程是否中断

7. **对象终结规则**：一个对象的初始化完成先行于发生它的finalize（）方法的开始。

8. **传递性**：如果操作A先行于操作B，操作B先行于操作C，那么操作A先行于操作C。

```java
int a = 0;
volatile boolean flag = false;

void write(){
    a = 1;//1
    flag = true;//2
}

void read() {
    if(flag){//3
        int i = 1;//4
    }
}
```
1. 由于程序次序规则，有(1)happens-before(2)
2. 由于volatile变量规则，有（2）happens-before（3）
3. 由于happens-before传递性规则，有（1）happens-before（4）
则上面的happens-before图示化如下：

![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jmm-volatile/happens-before.png)

在上图中，每一个箭头链接的两个节点，代表了一个happens before 关系。黑色箭头表示程序顺序规则；橙色箭头表示volatile规则；蓝色箭头表示组合这些规则后提供的happens before保证。

这里A线程写一个volatile变量后，B线程读同一个volatile变量。A线程在写volatile变量之前所有可见的共享变量，在B线程读同一个volatile变量后，将立即变得对B线程可见。

#### volatile写
> 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存。

以上面示例程序VolatileExample为例，假设线程A首先执行writer()方法，随后线程B执行reader()方法，初始时两个线程的本地内存中的flag和a都是初始状态。下图是线程A执行volatile写后，共享变量的状态示意图：

![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jmm-volatile/volatile-read.png)

如上图所示，线程A在写flag变量后，本地内存A中被线程A更新过的两个共享变量的值被刷新到主内存中。此时，本地内存A和主内存中的共享变量的值是一致的。

#### volatile读
> 当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

下面是线程B读同一个volatile变量后，共享变量的状态示意图：

![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jmm-volatile/volatile-write.png)

如上图所示，在读flag变量后，本地内存B已经被置为无效。此时，线程B必须从主内存中读取共享变量。线程B的读取操作将导致本地内存B与主内存中的共享变量的值也变成一致的了。

如果我们把volatile写和volatile读这两个步骤综合起来看的话，在读线程B读一个volatile变量后，写线程A在写这个volatile变量之前所有可见的共享变量的值都将立即变得对读线程B可见。

下面对volatile写和volatile读的内存语义做个总结：
- 线程A写一个volatile变量，实质上是线程A向接下来将要读这个volatile变量的某个线程发出了（其对共享变量所在修改的）消息。
- 线程B读一个volatile变量，实质上是线程B接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息。
- 线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程实质上是线程A通过主内存向线程B发送消息。

### volatile内存语义实现
下面，让我们来看看JMM如何实现volatile写/读的内存语义。

前文我们提到过重排序分为编译器重排序和处理器重排序。为了实现volatile内存语义，JMM会分别限制这两种类型的重排序类型。下面是JMM针对编译器制定的volatile重排序规则表：

**是否能重排序**

|| 普通读/写 | volatile读 | volatile写
---|:---:|:---:|:---:
普通读/写 | yes | yes | no
volatile读 | no | no | no
volatile写 | yes | no | no

举例来说，第三行最后一个单元格的意思是：在程序顺序中，当第一个操作为普通变量的读或写时，如果第二个操作为volatile写，则编译器不能重排序这两个操作。

从上表我们可以看出：
- 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
- 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。
- 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能，为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略：
- 在每个volatile写操作的前面插入一个StoreStore屏障。
- 在每个volatile写操作的后面插入一个StoreLoad屏障。
- 在每个volatile读操作的后面插入一个LoadLoad屏障。
- 在每个volatile读操作的后面插入一个LoadStore屏障。

### 内存屏障(Memory Barrier)
#### CPU的角度
其实这种问题不只是出现在Java上，毕竟一切的尽头都是机器指令，所以只要运行在计算机上都会有这种问题，所以其实指令集也针对乱序在多线程时出现的问题做出了拓展，这里我们以x86为例

- sfence: 内存写屏障，保证这条指令前的所有的存储指令必须在这条指令之前执行，并且在执行此条指令时把写入到CPU的私有缓存的数据刷到公有内存(以下均简称主存)
- lfence: 内存读屏障，保证这条指令后的所有读取指令在这条指令后执行，并且执行此条指令时，清空CPU的读取缓存，也就是说强制接下来的load从主存中取数据
- mfence: full barrier，代价最大的barrier，有上述两种barrier的效果，当然也是最稳健的的barrier
- lock: 这个是一种同步指令，也可以禁止lock前的指令和之后的指令重排序(有兴趣的同学可以去看一看这个指令，这个指令稍微复杂一些，可以实现的功能也很多，我就不贴了)，lock也许是很多JVM底层使用的指令
上述只是x86指令集下的相关指令，不同的指令集可能barrier的效果并不一样，fence和lock是两种实现内存屏障的方式(毕竟一个指令集很庞大)

#### Java的抽象
Java这个时候又来了一波抽象，他把barrier分成了4种

屏障类型 | 指令示例 | 解释
---|---|---
LoadLoadBarriers | Load1; LoadLoad;Load2 | 确保 Load1 数据的装载，之前于Load2 及所有后续装载指令的装载。
StoreStoreBarriers | Store1; StoreStore;Store2 | 确保 Store1 数据对其他处理器可见(刷新到内存)，之前于Store2 及所有后续存储指令的存储。
LoadStoreBarriers | Load1; LoadStore;Store2	Load1 | 数据装载，之前于Store2 及所有后续的存储指令刷新到内存。
StoreLoadBarriers | Store1; StoreLoad;Load2 | 确保 Store1 数据对其他处理器变得可见(指刷新到内存)，之前于Load2 及所有后续装载指令的装载。

StoreLoad Barriers 会使该屏障之前的所有内存访问指令(存储和装载指令)完成之后，才执行该屏障之后的内存访问指令。
注意，这是Java内存模型里的内存屏障，只是Java的规范，对于不同的处理器/指令集，JVM有不同的实现方式。

再次说明一下，这四个barrier是JVM内存模型的规范，而不是具体的字节码指令，因为你可以看到volatile变量在字节码中只是一个标志位，javap搞出来的字节码中并没有任何的barriers，只是说JVM执行引擎会在执行时会插一个对应的屏障，或者说在JIT/AOT生成机器指令的时候插一条对应逻辑的barriers，说句人话，这个barrier不是javac插的！所以你通过javap看不到，如果想要看到volatile的作用，可以把字节码转成汇编(很多很多)，具体指令如下

```
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly [ClassName]
```

