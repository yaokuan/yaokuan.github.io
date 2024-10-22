---
layout:     post
title:      深入反射解密JVM元空间OOM
date:       2023-12-22 22:24:00
categories: JVM
tags:       Java 反射 类加载 JDK 元空间 OOM 软引用 并发线程安全
---

* 文章目录
{:toc}









# 1 背景
> **标签： Java 反射 类加载 JDK 元空间 OOM 软引用 并发线程安全**

笔者在生产实践中碰到过很多OOM相关的问题，其中因为元空间引发的OOM问题不在少数，大多数堆内存的问题结合JVM参数、GC日志、堆dump分析还是比较容易有清晰的方向的，而元空间的问题往往是比较难定位的。因此笔者借助一次元空间OOM的排查经历来详细了解下什么是元空间，为什么元空间会OOM以及如何避免。为了让读者能够快速有个大概的印象，先介绍下本文的写作思路，本文会从问题描述，介绍元空间，分析案例现象引出频繁类加载问题，在过程中会了解反射执行过程，初识`GeneratedMethodAccessor`类，分析`GeneratedMethodAccessor`类重复加载的原因，进行实验验证，JVM参数调优实践。这个过程中会涉及到反射调用的实现细节，膨胀（inflation）机制及其解决的问题，反射和软引用的关系，反射过程中的线程安全，JVM参数调优等知识点，本文主要围绕JDK 8来展开排查和调优过程，也会介绍JDK 7/11/17/21等版本的反射实现和调优思路。

# 2 一个线上元空间OOM案例

## 2.1 简述线上元空间OOM现象

线上某个应用突然开始不停地有 `java.lang.OutOfMemoryError: Metaspace` 告警，看元空间占用内存一直在增长的，一般来说元空间的增长都是因为类加载导致的。继续看了下应用的 `jvm.classloading.loaded.count` 指标趋势图，基本一直在增长直到下一次服务发布或者是Full GC时才会降下来一些，如果应用间隔了一段时间没有发布就会频繁Full GC直到元空间OOM告警，为了搞清楚元空间OOM的根因，首先得了解下什么是元空间。

![image](/img/20231222-1.jpg)
<center style="color:gray">图1 应用类加载数量趋势图</center>

## 2.2 元空间介绍

元空间是JDK 8才出现的概念，在JDK 8之前叫做永久代，它们都属于方法区，元空间、永久代、方法区看似同一个东西实际又有所区别。方法区是Java虚拟机的一个规范，所有虚拟机的实现都必须遵循规范，常见的虚拟机如`HotSpot`、`JRockit（Oracle）`、`J9（IBM）`都有对方法区不同的实现。我们通常使用的JDK都是由`Sun JDK`和`OpenJDK`所提供的，这也是应用最广泛的版本，而该版本使用的VM就是`HotSpot VM`，所以我们一般讨论就限定在`HotSpot VM`的范围。而元空间和永久代是`HotSpot VM`对方法区不同的具体实现，其区别包括：

实现版本不同。永久代是JDK 7之前的实现，元空间是JDK 8及之后的实现。

存储位置不同。永久代占用虚拟机分配的内存，图2中看永久代和堆是连在一起的，表明这两块空间在物理内存上是连续的，但在逻辑上是独立的，而元空间使用本地内存不受虚拟机内存限制，和堆内存在物理内存上不连续了。

存储内容不同。永久代存储类信息、字面量、类静态变量和符号引用，而元空间只存储类信息，实际上移除永久代的工作从JDK 7就开始了，符号引用转移到了`Native heap`，字面量和类静态变量转移到了`Java heap`。

![image](/img/20231222-2.jpg)
<center style="color:gray">图2 元空间和永久代的区别</center>

那为什么JDK 8要新引入元空间来替换永久代？

字符串存在永久代中，程序中大量使用字符串常量容易出现性能问题和内存溢出。

类和方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易导致老年代溢出。

永久代会为GC带来不必要的复杂度，并且回收效率偏低。

JDK 8实际上是`HotSpot`与`JRockit`合并的，只是产品名字还是叫`HotSpot VM`，合并就需要移除HotSpot特有的永久代，新引入元空间给新的`HotSpot VM`。

## 2.3 元空间OOM问题初次分析

那我们知道元空间主要就是存储类信息，再结合从2.1节观察 `jvm.classloading.loaded.count` 指标的增长趋势，我们基本可以判断这次元空间OOM是因为有频繁的类加载行为，我们可以理解在应用启动时会有类加载，但为什么应用在运行时还会加载这么多类需要重点排查下，JVM可以打印类加载和卸载的日志，需要添加下面的参数

- `-XX:+UnlockDiagnosticVMOptions` 允许查看Metaspace加载的类信息和元信息使用情况

- `-XX:+TraceClassUnloading -XX:+TraceClassLoading` 打印JVM类加载和卸载的信息
- 
添加参数重启应用后查看日志可以发现在应用运行中仍有大量类似`sun.reflect.GeneratedMethodAccessor`的类被加载

```shell
...此处省略10000条
[Loaded sun.reflect.GeneratedMethodAccessor6186 from __JVM_DefineClass__]
...
```

看类的命名能知道是JDK中自带的类，虽然在JDK源码中搜不到这个类（3.1.2小节会解释为什么搜不到），但通过`sun.reflect`这个包名我们也能大概猜到这是和反射相关的类，但不知道是哪里用到了反射，为了进一步分析就需要借助反编译工具

### 2.3.1 通过反编译工具分析大量加载类的来源

前面加了类加载卸载日志应用重启后不到10分钟，加载`sun.reflect.GeneratedMethodAccessor`类似的类就多达1600+，这是比较快的增长速度

```shell
[localhost ~]$ jcmd 158750 GC.class_histogram > class_histogram_20190804_14_57
[localhost ~]$ cat class_histogram_20190804_14_57 |grep "sun.reflect.GeneratedMethodAccessor"|wc -l
1674
```

但为什么类持续在加载却并没有看到大量类卸载的日志，而且元空间占用也是随着类加载趋势一直在上涨。这里思考一下为什么加载的这些类不能被回收？我们知道类被回收需要满足下面三个条件，因为反射是由应用类加载器加载，很明显下面的第2点就不满足，所以反射过程中动态加载的这些类无法被回收，类加载数量持续增长。
1.该类所有的实例都已经被回收，也就是java堆中不存在该类的任何实例。
2.加载该类的ClassLoader已经被回收（双亲委派机制，判断两个类相同的前提是加载类的classLoader相同）。
3.该类对应的`java.lang.Class`对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法。
为了进一步分析这些`GeneratedMethodAccessor`类是在哪里反射调用的，我们借助开源工具dumpclass下载这些类再通过反编译可以获取更多详细的信息

```shell
#下载dumpClass的jar包
wget http://search.maven.org/remotecontent?filepath=io/github/hengyunabc/dumpclass/0.0.2/dumpclass-0.0.2.jar -O dumpclass.jar
#dumpClass需要设置JAVA_HOME的环境变量
export JAVA_HOME=/usr/local/java8
#将GeneratedMethodAccessor开头的类下载到根目录
java -jar /dumpclass.jar -p [PID] -o / sun.reflect.GeneratedMethodAccessor*
#在目录下可以看到GeneratedMethodAccessorXXXX.class
cd /sun/reflectsun.reflect.GeneratedMethodAccessor*
#随便找一个反编译出来
javap -verbose GeneratedMethodAccessor999
```

查看反编译class文件找到反射执行的位置（在class文件中找到`invokevirtual`关键字，对应的路径则为反射生成`GeneratedMethodAccessor`的业务代码）进行分析发现大量调用来源于业务中bean的`Getter/Setter`方法，通过脚本统计反编译类的`invokevirtual`属性发现存在同一个Getter方法生成多个`GeneratedMethodAccessor`的情况。
![image](/img/20231222-3.jpg)
<center style="color:gray">图3 反编译字节码类查看业务调用源头</center>

### 2.3.2 初步问题分析的结论引出重复类加载的疑点

所以关于频繁的`GeneratedMethodAccessor`类加载我们初步得出下面几个结论：
1.生成的`GeneratedMethodAccessor`类大多是正常业务代码相关的，各种类都有并没有某一特定`Getter/Setter`方法生成大量的`GeneratedMethodAccessor`类。
2.同一个业务类的`Getter/Setter`方法存在生成多个`GeneratedMethodAccessor`类的情况，也就是可能会出现重复加载`GeneratedMethodAccessor`类的情况。
第2点结论比较反直觉，我们一般认为对于某个反射方法已经是非常明确的了，所生成的字节码类应该只有唯一的一个且是可以复用的，如果是这样的话因为业务类的数量是固定的，那么对应反射方法的动态类加载数量理论上也应该是固定的。但实际上反射相关的类是持续增长且有重复加载的现象，也正是因为重复加载才会出现类数量持续增加最终导致元空间OOM的情况。所以为什么会出现重复加载的情况应该是问题的关键，我们带着疑问继续往下看。

# 3 初识反射相关类GeneratedMethodAccessor（GMA）

> **注：以下内容提到的GMA为GeneratedMethodAccessor类的缩写**

## 3.1 了解反射执行过程及GMA类在反射中的作用

首先我们需要了解反射调用执行大概的过程，先看一个简单的反射例子，下面是通过反射调用拿到了person的id，首先要通过Person类获取对应的getId方法，然后执行对应的反射方法`method.invoke()`。

![image](/img/20231222-4.jpg)
<center style="color:gray">图4 一个反射例子</center>

这里`java.lang.Class#getMethod`和`java.lang.reflect.Method#invoke`是比较关键的两个方法，为了大家可以更好地理解反射的执行流程，我们通过下面的时序图大致画出如图4例子反射调用的执行过程，让大家先有一个大概的印象，其中有许多细节在下文中慢慢展开。

![image](/img/20231222-5.jpg)
<center style="color:gray">图5 反射调用执行时序图</center>

### 3.1.1 通过Class如何获取反射方法

我们先来看下是怎么找到方法的，顺着`java.lang.Class#getMethod`往下会发现查找方法都会走到`java.lang.Class#privateGetDeclaredMethods`内部，`publicOnly`参数是用来控制是否只返回声明为public的方法，这里以图4为例publicOnly参数为true，对应返回`declaredPublicMethods`，方法内会首先判断`ReflectionData`是否为空，这是一个Class内部的软引用相当于本地缓存的作用，图6标红第3处会判断缓存内部的`declaredPublicMethods`是否为空，如果缓存内非空优先返回缓存内的，否则调用JNI获取类声明的public方法并放入缓存中。

![image](/img/20231222-6.jpg)
<center style="color:gray">图6 获取Class内的Method</center>

再看下方法的缓存是在`java.lang.Class#reflectionData`方法获取的，前面有提到`reflectionData`相当于本地缓存（图7标红1处是在Class内部声明为软引用的局部变量），软引用常被用来当做缓存使用是因为其特殊的回收机制（3.2.3小节有介绍）。useCaches参数初始化为true，在图7标红3处可以通过系统变量`-Dsun.reflect.noCaches`关闭，所以默认开启缓存并返回缓存对象（不存在则新建一个空缓存返回）。

![image](/img/20231222-7.jpg)
<center style="color:gray">图7 Class内软引用的使用</center>

通过上面的步骤就找到了Class声明的所有public方法，之后就要从中找到我们需要的那一个，在`java.lang.Class#searchMethods`方法中会通过方法名找到完全匹配的方法Method，然后在图8标红2处复制一个Method对象返回，复制过程使用的是`java.lang.reflect.Method#copy`方法，关于Method的复制我们后面还会再提到，这里只要暂时先知道方法的复制会深拷贝一个Method对象返回即可。至于为什么要每次进行拷贝而不能直接用找到的method呢？笔者认为这里至少有两个原因

因为同一个方法的不同实例化对象中，虽然大部分属性都相同，如方法名、入参、返回值等，但有一个属性是不同的，它就是 override 属性默认为 false，控制是否能够访问私有方法，如果当前方法对象是私有方法，默认是不可访问的，可以通过 `setAccessible(true)` 进行修改 ，复制是为了避免对原方法override的修改。

原有的method是一个全局变量，如果直接使用了原有的method，可能会对该method有一定的修改，该method会被多个线程使用，从而导致某个线程某处对method修改后，对别的线程产生影响，getMethod方法每次都会copy出一个新的副本method，可以减少同一线程内和不同线程间method修改产生的影响。

![image](/img/20231222-8.jpg)
<center style="color:gray">图8 通过方法名搜索Method</center>

至此我们就通过Class获取到了对应的方法Method对象，这个Method对象是对原对象的一个深拷贝，之后就要进行反射调用了。

### 3.1.2 Method执行反射调用的过程分析

#### 3.1.2.1 获取MethodAccessor流程分析

`java.lang.reflect.Method#invoke`方法是反射执行调用的关键，在图9标红1可以看到invoke()内部声明了一个局部变量ma它是MethodAccessor接口的一个实例化对象，而MethodAccessor接口定义了反射具体行为，`Method.invoke`的真正逻辑在图9标红3处的 
`ma.invoke(obj, args)`中，因为实际执行逻辑是委托给MethodAccessor进行的，所以这里我们首先要搞清楚MethodAccessor是如何生成的以及如何利用MethodAccessor来执行反射调用。

![image](/img/20231222-9.jpg)
<center style="color:gray">图9 invoke方法的流程</center>

在图9标红2处可以看到获取MethodAccessor实例并赋值给ma，进入acquireMethodAccessor()方法内部，在图10中有明确的`if...else`两条代码分支，明确了两种不同的MethodAccessor实现，这两种实现都非常重要。if内部是复用root的MethodAccessor，root是Method内的局部变量（图10标红5），类型也是Method，前面有提到我们获取的Method对象都是深拷贝的，这个root就是指向的原Method对象，而else内部是新建MethodAccessor的方式。前面图6标红2的JNI调用获取的Method的methodAccessor默认值是null的，Method的methodAccessor变量被设计成懒加载的（图10标红4），笔者推测这是因为虚拟机不知道方法何时被反射调用，所以为了节省内存开销设计成了懒加载。所以默认第一次反射调用会进入到图10标红2执行新建流程。

![image](/img/20231222-10.jpg)
<center style="color:gray">图10 获取和生成MethodAccessor流程</center>

先看下新建流程里，会通过`sun.reflect.ReflectionFactory#newMethodAccessor`去创建methodAccessor对象，图11中有两个条件来控制使用哪种方式创建MethodAccessor对象

1. `noInflation`。是否关闭inflation机制[^1]，默认情况下noInflation为false，若设置为true则始终使用字节码方式生成GMA（可通过JVM参数 `-Dsun.reflect.noinflation`来设置）。

2. `!ReflectUtil.isVMAnonymousClass`。这个条件是判断方法所属的类非匿名类，因为匿名类不支持字节码的方式，同样的判定条件在NativeMethodAccessorImpl也有原因详见注释（图12）。

因为noInflation参数默认为false，会进入到else内部生成`NativeMethodAccessorImpl`对象并给`DelegatingMethodAccessorImpl`代理执行。

![image](/img/20231222-11.jpg)
<center style="color:gray">图11 新建methodAccessor流程</center>
![image](/img/20231222-12.jpg)
<center style="color:gray">图12</center>

#### 3.1.2.2 关于MethodAccessor接口和它的实现类

`MethodAccessor`接口定义了反射主要的行为，图11中可以看到 `NativeMethodAccessorImpl`（本地方法调用类）交给了`DelegatingMethodAccessorImpl`（代理类）代理执行，除这两种外还有 `MethodAccessorImpl`（抽象实现类）和`GeneratedMethodAccessor`[^2]（字节码类）。我们会发现在JDK源码中根本找不到`GeneratedMethodAccessor`这个类，是因为这是ASM字节码生成的类，看源码字节码生成方法返回的是`MagicAccessorImpl`，这是个什么类？

原本Java的安全机制使得不同类之间不是任意信息都可见，但JDK里面专门设了个`MagicAccessorImpl`标记类开了个后门来允许不同类之间信息可以互相访问（由JVM管理），所以`MethodAccessorImpl`类继承`MagicAccessorImpl`类后可以访问字节码生成的`GeneratedMethodAccessor`类信息。

```java
/* <P> MagicAccessorImpl (named for parity with FieldAccessorImpl and
    others, not because it actually implements an interface) is a
    marker class in the hierarchy. All subclasses of this class are
    "magically" granted access by the VM to otherwise inaccessible
    fields and methods of other classes. It is used to hold the code
    for dynamically-generated FieldAccessorImpl and MethodAccessorImpl
    subclasses. (Use of the word "unsafe" was avoided in this class's
    name to avoid confusion with {@link sun.misc.Unsafe}.) </P>
    <P> The bug fix for 4486457 also necessitated disabling
    verification for this class and all subclasses, as opposed to just
    SerializationConstructorAccessorImpl and subclasses, to avoid
    having to indicate to the VM which of these dynamically-generated
    stub classes were known to be able to pass the verifier. </P>
    <P> Do not change the name of this class without also changing the
    VM's code. </P> */
class MagicAccessorImpl {
}
```

所以再来看一下图13的UML，我们可以知道`MethodAccessor`接口有native和字节码两种具体实现类，并使用代理模式（将`NativeMethodAccessorImpl`和`GeneratedMethodAccessor`交给`DelegatingMethodAccessorImpl`统一进行代理）进行调用，`DelegatingMethodAccessorImpl`只负责转发，实际负责执行的是`NativeMethodAccessorImpl`和`GeneratedMethodAccessor`。
![image](/img/20231222-13.jpg)
<center style="color:gray">图13 MethodAccessor接口UML</center>

生成完`NativeMethodAccessorImpl`和`DelegatingMethodAccessorImpl`后，就会将新生成的methodAccessor赋值给Method并同步赋值给root（图10标红3），`java.lang.reflect.Method#methodAccessor`被声明为volatile的，这样只要有一处设置过就可以实现methodAccessor的复用了。
![image](/img/20231222-14.jpg)
<center style="color:gray">图14 设置root复用methodAccessor</center>

那么下一次反射调用执行时，看图10标红1处的root是Method内部声明的一个Method类型的局部变量，在前面图7标红5处我们有提到获取的方法都是通过`Method.copy()`复制的，再看图15标红1在复制时会先深拷贝Method原对象生成一个Method新对象（包括方法名、参数、返回值等），图15标红2处新对象的root是一个到原对象的引用，然后图15标红3处复用原对象的methodAccessor对象（图10的root声明的注释也有强调说明root的存在是为了复用methodAccessor对象）。所以图10标红1这里当root不为null时就直接复用root的methodAccessor返回，不需要重新创建MethodAccessor。所以可以理解之后所有获取的Method新对象的root都是指向原对象，那么methodAccessor也是原对象的methodAccessor，都是同一个methodAccessor对象。
![image](/img/20231222-15.jpg)
<center style="color:gray">图15 复制Method的流程</center>

#### 3.1.2.3 分析MethodAccessor的执行过程

到这里我们就获取到`MethodAccessor`了，前面提到初次返回的是`DelegatingMethodAccessorImpl`的实例，它是对`NativeMethodAccessorImpl`实现类的代理，所以在`method.invoke()`内执行的反射调用最终是交给`NativeMethodAccessorImpl`执行，图16的标红的几点都比较重要，我们按标红序号展开来说一下。

这里是`DelegatingMethodAccessorImpl`的部分实现，代理类此时是对`NativeMethodAccessorImpl`的代理，所以这里的delegate是`NativeMethodAccessorImpl`的实现，最终会由`NativeMethodAccessorImpl`来执行反射调用。

反射执行过程中会判断`++numInvocations > ReflectionFactory.inflationThreshold()`，阈值inflationThreshold是根据JVM启动时 `-Dsun.reflect.inflationThreshold` 参数进行设置的，默认值是15，关于这个判断阈值的机制我们后面会详细展开，这里暂时只需要了解到阈值才会进入生成GMA的流程中。

先通过ASM生成GMA字节码类，这里的parent就是代理类，将GMA类的实例委托给代理类，那么后续标红1处的反射调用就是通过GMA类来执行了。

没有超过阈值就会通过JNI调用执行。
![image](/img/20231222-16.jpg)
<center style="color:gray">图16</center>

### 3.1.3 GeneratedMethodAccessor类的来源

到这里反射执行的过程我们大概清楚了，从前面的分析流程中我们发现有两处是通过字节码生成MethodAccessor的（查看代码引用也只有这两处在使用MethodAccessorGenerator来生成MethodAccessor）

1.第一处是在`sun.reflect.ReflectionFactory#newMethodAccessor`方法中，因为noInflation参数默认为false不会进入到字节码的代码中，所以暂不讨论。

2.第二处就是在图16标红3处，`sun.reflect.MethodAccessorGenerator#generateMethod`方法中可以看到是通过ASM字节码框架动态生成了Java字节码文件，继续跟进到generate()方法中，参数是方法相关的各种信息，返回值的就是我们前面提到的MagicAccessorImpl类。继续往下会发现类名是在generateName方法生成，isConstructor入参为false所以会进入else代码块（generate方法不仅用于MethodAccessor的生成，还可以用于构造方法描述类的生成），最终生成类似`sun/reflect/GeneratedMethodAccessor+`数字的类名（图17标红3），这个类名正式我们前面排查类加载数量持续增长的GMA类。
![image](/img/20231222-17.jpg)
<center style="color:gray">图17</center>
![image](/img/20231222-18.jpg)
<center style="color:gray">图18</center>

到这里关于GMA类和反射的关系就很清晰了，类似`sun/reflect/GeneratedMethodAccessor123`的类我们统称GMA类，GMA类是通过ASM字节码生成的MethodAccessor的实现类，主要作用是执行对真实方法Method的反射调用。GMA类不是默认生成的，而是需要NativeMethodAccessorImpl的实现下超过一定的阈值，才会优化成使用字节码。所以我们现在弄清楚了为什么会产生GMA类以及产生的时机，但是按前面分析流程methodAccessor都是会复用的，理论上一个方法最多只会生成一个GMA类，而这与我们在2.3节问题描述中提到的一个getter/setter方法生成多个GMA类的结论是矛盾的，所以下一个更重要的问题是为什么GMA类会重复生成？以及在什么条件下会重复生成？

## 3.2 GeneratedMethodAccessor重复类生成的原因

前面我们分析了字节码生成都是有相应条件的，既然要深入GMA重复类生成原因，我们就要先剖析一下影响GMA字节码类生成的inflation机制。前面我们了解到反射中的invoke可以通过JNI调用或者是字节码来完成，而inflation机制就是以提升反射执行速度为目的的优化，inflation机制主要是通过下面两个参数实现的。

### 3.2.1 反射调用膨胀机制（inflation机制）

Java虚拟机会首先使用JNI存取器，然后在访问了同一个类若干次后，会改为使用Java字节码存取器。 这种当Java虚拟机从JNI存取器改为字节码存取器的行为被称为膨胀（Inflation）

前面我们提到跟GMA类生成相关的地方有两处，而这两处都有inflation机制相关的参数来控制。

noInflation参数，在`sun.reflect.ReflectionFactory#newMethodAccessor`处

这个方法是创建MethodAccessor对象，通过noInflation参数控制实现方式，如果该参数为true则在newMethodAccessor方法中就通过字节码会直接创建GMA类来实现invoke方法，如果为false就会走默认的DelegatingMethodAccessorImpl代理NativeMethodAccessorImpl实现类。因为参数noInflation默认为false（可通过-Dsun.reflect.noinflation进行设置参考图20）所以默认这里不会生成GMA类。
![image](/img/20231222-19.jpg)
<center style="color:gray">图19</center>

inflationThreshold参数，在`sun.reflect.NativeMethodAccessorImpl#invoke`处

这个方法是NativeMethodAccessorImpl实现反射调用的相关代码，inflationThreshold参数就是控制inflation机制的阈值，当同一个Method的invoke被调用次数未超过阈值时是JNI调用，超过阈值时生成GMA类优化反射调用。inflationThreshold默认值为15（可通过-Dsun.reflect.inflationThreshold进行设置参考图x.x），该值越小越容易触发GMA类的生成，相反设置得越大就越不容易触发GMA类生成。
![image](/img/20231222-20.jpg)
<center style="color:gray">图20</center>

通过上面的分析，已经搞清楚了反射的inflation机制，知道了GMA类生成的时机，但是为什么虚拟机不设计成始终使用JNI来执行反射调用呢？设计这么复杂的GMA类和inflation机制是什么原因？

为什么反射需要字节码和native两种方式以及inflation的好处

先来说一下为什么需要字节码和native两种实现方式，native调用和字节码调用的区别包括

1.Java 字节码调用效率比 native 调用要快 20 倍以上
2.Java 字节码加载时要比 native 多消耗 3~4 倍资源
3.Java字节码在初始化时需要较多时间和较多资源，但持续运行因为热点代码优化会比native更快，使用Java字节码可以避免JNI为了维护OopMap进行封装/解封装而带来的性能损耗
4.native版本正好相反，启动时直接JNI调用较快，但持续运行因为跨越native边界会导致JIT优化不到，长期看性能不如Java字节码

因为字节码和native在性能和开销上的差异，所以虚拟机设计了inflation机制，这个机制跟HotSpot命名有点异曲同工之妙。我们都知道虚拟机通过解释器（Interpreter）来执行字节码文件，且虚拟机内有计数器来统计代码的调用次数，当虚拟机发现某个方法或代码块的运行特别频繁时，就会把这些代码认定为“热点代码”（Hot Spot Code），这也就是HotSpot虚拟机的命名来源。热点代码会通过JIT编译生成机器码，比解释器逐条执行的效率会高很多。至于为什么HotSpot不使用JIT全程编译，也是在性能和开销上取的折中，想详细了解的可以参考下为什么 JVM 不用 JIT 全程编译？-- R大。
![image](/img/20231222-21.jpg)
<center style="color:gray">图21 HotSpot简介图</center>

### 3.2.2 生成GMA类过程中可能因为并发导致重复的原因分析

前面有分析到默认情况下GMA生成只有一个地方，那就是在`sun.reflect.NativeMethodAccessorImpl#invoke`处（参考图20），分析这块代码不难发现在`++numInvocations > ReflectionFactory.inflationThreshold`的判断条件处并没有加任何同步机制，虽然Method是复制出来的，多个线程持有的Method都是不同的，但methodAccessor是复用的，也就是多个线程反射调用使用的是同一个`NativeMethodAccessorImpl`的实例，而`numInvocations`是`NativeMethodAccessorImpl`的局部变量，在并发环境下可能有线程不安全的问题，可能会导致多个线程同时进入条件为true的代码块中，生成多个GMA类。而我们的线上应用一般都是并发环境，如果有一个反射方法的`invoke()`调用比较集中，那就可能在inflation机制阈值判断时，有多个线程进入GMA类的生成代码中。

其实method.invoke的过程中未加同步机制是JDK 8设计如此，除了上面提到的地方外，还有另外一处在获取`MethodAccessor`的方法`acquireMethodAccessor()`方法注释处也有给出说明。在并发环境下可能有多个线程进入else代码块生成MethodAccessor对象，所以可能生成多个`NativeMethodAccessorImpl`和`DelegatingMethodAccessorImpl`对象，然后会有一个线程去`setMethodAccessor(tmp)`，因为Method内的MethodAccessor被声明为volatile的立马对其他线程可见，后续就会复用MethodAccessor对象。所以正如下图注释中解释的，可能会生成多个MethodAccessor对象但不影响反射调用执行结果，且这种不加同步的实现方式会具有更好的扩展性。
![image](/img/20231222-22.jpg)
<center style="color:gray">图22</center>

### 3.2.3 生成GMA类过程中可能因为软引用回收导致重复的分析

按3.2.2节分析因为并发线程安全的问题导致GMA类的增长，虽然会导致GMA类重复生成，但这个应该增长的量应该是有限的，因为一旦这个方法生成了对应的GMA类后续都会一直复用，但实际过程中会发现持续的类加载，我们再看下图20生成GMA的流程，因为前面我们已经判断到GMA类只有在这个地方生成，稍加分析不难发现这个if判断条件里最有可能变化的就是计数器`numInvocations`，因为默认要累加到15次才会生成GMA类，那持续有类加载的现象很可能是因为计数器清零了因为一旦计数器重置就可能再次生成GMA类，导致GMA类持续增长，那么计数器是否会重置以及为什么会重置？

因为`numInvocations`是`NativeMethodAccessorImpl`的局部变量，内部没有其他地方修改，那么最有可能就是`NativeMethodAccessorImpl`实例变了，再联想到前面分析的所有复制出来的Method都是指向同一个`methodAccessor`，那么不能复用的情况可能是`methodAccessor`不能再被引用到了。这里面有一个值得注意的点，`ReflectionData`的对象被设计为一个软引用`SoftReference`来充当缓存减少JNI调用，而软引用的回收时机总得来说有两个

1.内存充足，软引用的回收使用LRU算法，存活时间越长该软引用对象越容易被回收，可以通过`-XX:SoftRefLRUPolicyMSPerMB`参数控制回收的时机

2.内存不足，在OOM或者快要OOM时软引用的回收算法是全量回收

所以如果应用持续运行，跨越了多个软引用回收周期，那么复用的这个`methodAccessor`可能随着原Method对象被回收，下次再通过JNI调用生成的就是一个全选的Method对象以及一个全新的`methodAccessor`对象，当然`numInvocations`计数器也会被重置，持续反射调用很可能再次触发阈值生成新的GMA类。

## 3.3 通过实验验证并发和跨软引用周期导致GMA类重复

对反射的执行过程有了一个了解后，我们大概能知道这里面为啥会重复生成GMA类了，为此我们专门设计对照组和实验组实验来实际验证下前面的结论是否符合预期。

### 3.3.1 正常实验流程（无并发，单个软引用周期）

按正常实验流程GMA生成次数都是符合我们预期的，不会有重复GMA生成。

| 场景描述                      | GMA类数量 | 源码截图                       |
|-----------------------------|----------|--------------------------|
| 单线程一个method.invoke15次   | 0        | ![image](/img/20231222-23.jpg) <br><center style="color:gray">图23</center> |
| 单线程一个method.invoke16次   | 1        | ![image](/img/20231222-24.jpg) <br><center style="color:gray">图24</center> |
| 单线程两个method.invoke各10000次 | 2        | ![image](/img/20231222-25.jpg) <br><center style="color:gray">图25</center> |


### 3.3.2 有重复GMA类生成的流程

经过上面软引用和并发安全的分析，我们特地设计了三组实验，参考上面正常对照组的结果对比实验充分地说明了GMA重复类的生成跟并发和软引用回收是相关的。

| 场景描述                          | GMA类数量 | 截图                       | 原因分析                                                                                                      |
|---------------------------------|----------|--------------------------|-----------------------------------------------------------------------------------------------------------|
| 无并发多周期                        | 2        | ![image](/img/20231222-26.jpg) | method1置null方便软引用回收，若不置为null，则在分配大对象回收软引用时，因为有强引用而不能回收，无法复现多个GMA的场景。分配大对象，触发分配失败的Full GC，系统回收软引用后打印为null。调用invoke 15次以上触发阈值，会生成新的GMA类。 |
| 并发单周期                          | 20       | ![image](/img/20231222-27.jpg) | 在第一个线程超过阈值进入生成字节码到第一个ma生成并赋值给method对象过程中的其他线程都可能会重复生成GMA，过程如图29。![image](/img/20231222-28.jpg) |
| 并发多周期                          | 28       | ![image](/img/20231222-29.jpg) | 第一个软引用周期，预期最多生成20个，实际生成20个GMA类，之后不再新生成。中间分配大对象手动触发一次软引用回收，打印为null（图30）。第二个软引用周期，预期最多生成20个，实际生成8个GMA类，之后不再新生成。                   |


结合源码和实验分析有重复GMA类生成的主要原因有两个：

1.生成GMA的过程不是线程安全的，在并发的环境下可能会有多个线程重复生成反射方法的GMA类。

2.反射方法的methodAcessor为软引用会被JVM GC回收，回收后在下次method.invoke()执行会重置计数器，invoke次数超过阈值时会重新生成新的GMA类。

## 3.4 问题梳理总结

有了前面的源码分析和实验结果验证，持续加载GMA类的过程就比较清晰了，下图直观地描述了重复GMA类加载的过程，对于重点地方我们用文字简单来总结下
![image](/img/20231222-30.jpg)
<center style="color:gray">图30</center>

1.每个Class都有一个软引用类型的`ReflectionData`数据结构，设计为软引用是作为缓存来加速反射的调用，`ReflectionData`中存储着当前类中的字段、方法等元信息，通过JNI获取的字段、方法等元信息会放到`ReflectionData`中。

2.获取反射方法时通过方法名匹配到对应的Method对象，出于线程安全考虑，每个新获取的method对象都是对原对象的深拷贝（字段、方法等），同时会复用`methodAccessor`，原Method和所有复制出的Method都指向同一个`methodAccessor`。

3.`methodAccessor`在一个软引用周期内只会初始化一次，作为实现类`NativeMethodAccessorImpl`的对象中`numInvocations`会从零开始计数，与`inflationThreshold`进行比较，此时如果有并发等情况出现，多个线程持有相同的`methodAcessor`时，调用invoke都会使`numInvocations`自增，此时因为线程不安全如果有多个线程同时判断`numInvocations`超过阈值时就都会去创建一个GMA对象，GMA类就会重复生成。

4.如果跨多个软引用周期，当Class的软引用被回收后，下次获取Method时就会通过JNI生成一个新的Method对象，然后在执行反射方法调用时再生成一个新的`methodAccessor`对象，那新的`NativeMethodAccessorImpl`的对象中`numInvocations`会清零，超过阈值就会生成GMA类，此时如果有并发情况还可能生成更多的GMA类。

5.步骤3和步骤4会随着应用生命周期持续进行，直到元空间内存溢出触发OOM。

# 4 解决方案

到这里问题基本明确了，因为线上应用使用了Spring框架的`org.springframework.beans.BeanUtils#copyProperties`进行Bean到DTO的转换，框架底层是通过反射来实现对象的复制，因为线上应用涉及到比较多的业务类，在并发访问环境下跨多个软引用周期就会生成重复的GMA类直到元空间内存不足触发了元空间OOM，对此我们可以针对性地去优化。

## 4.1 相关JVM参数调优方案

### 4.1.1 元空间大小相关JVM参数

所有系统都应该注意元空间的大小设置，对于64位JVM来说元空间的默认初始大小是20.75MB，默认的元空间的最大值是无限，`-XX:MetaspaceSize=N和-XX:MaxMetaspaceSize=N`，由于元空间大小的扩容中需要`Full GC`，这是非常昂贵的操作，如果应用在启动的时候发生大量Full GC，通常都是由于永久代或元空间发生了大小调整，基于这种情况，一般建议在JVM参数中将`MetaspaceSize`和`MaxMetaspaceSize`设置成一样的值，下面介绍一个这两个参数的含义：

1.`MetaspaceSize`：设置Metaspace扩容时触发`Full GC`的初始化阈值，也是最小的阈值；`Metaspace`由于使用不断扩容到`-XX:MetaspaceSize`参数指定的量，就会发生`Full GC`，且之后每次`Metaspace`扩容都可能会发生`Full GC`。

2.`MaxMetaspaceSize`：设置可以分配给类元空间的最大本地内存，默认情况下大小不受限制。

综上，调大元空间的大小可以缓解类加载导致元空间OOM的问题，但无法完全避免并发和跨软引用周期导致重复加载GMA类。

>最佳实践：建议将`MetaspaceSize`和`MaxMetaspaceSize`设置成一样的值避免扩容，对于8G物理内存的机器来说一般建议将这两个值都设置为256M（读者可以根据实际情况调整）

### 4.1.2 inflationThreshold参数

JNI调用优化为GMA字节码的反射调用次数阈值可以通过参数`-Dsun.reflect.inflationThreshold`调整，因为`inflationThreshold`默认值是15，如果将`inflationThreshold`适当调大可以尽可能减少动态反射类的生成（如需关闭inflation，在Oracle JVM中没有直接关闭参数需要设置为int最大值，在IBM JVM中可以通过设置为0来关闭inflation）。但是这个动作需要谨慎，毕竟JNI的速度相对字节码还是很慢，如果应用对时间很敏感，不建议直接将`inflationThreshold`调为最大值，可以配合调整元空间的大小的同时将`inflationThreshold`调整到适当的值。

>最佳实践：默认值15，这个参数可以结合实际情况适当调大，如果要关闭`inflation`可以设置为`Integer.MAX_VALUE`，但关闭后会失去JIT反射性能优化，需要慎重操作

### 4.1.3 noInflation参数

还可以通过`-Dsun.reflect.noInflation=false（默认是false）`来关闭反射的优化，但仍然会有inflation机制生成字节码类。

>最佳实践：默认值false，一般不建议开启

### 4.1.4 SoftRefLRUPolicyMSPerMB参数

笔者在实际解决问题的过程中也调整过`SoftRefLRUPolicyMSPerMB`参数，实际调优发现效果并没有达到预期，这里也一并说明下。软引用对象在GC的时候到底要不要被回收是通过如下的一个公式来判定的，类似一个LRU缓存策略

`clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB`

1.`clock - timestamp` 代表了一个软引用对象他有多久没被访问过了

2.`freespace` 代表JVM中的空闲内存空间

3.`SoftRefLRUPolicyMSPerMB` 代表每一MB空闲内存空间可以允许SoftReference对象存活多久

`SoftRefLRUPolicyMSPerMB`参数默认值是0所以公式的右半边是0，就导致所有的软引用对象，比如JVM生成的字节码类，刚创建出来就可能在下一次GC时被回收掉，所以也可以通过调大这个参数来降低软引用被回收的频率。
![image](/img/20231222-31.jpg)
<center style="color:gray">图31 类加载与软引用回收流程</center>

`-XX:SoftRefLRUPolicyMSPerMB=1000（单位：毫秒/MB）`

比如我们调整参数为1000，假如新生代还剩1800MB，那么软引用对象可以生存的时间实际就是`1800MB * 1s/MB ≈ 30分钟`

要注意调大这个参数只能说是缓解跨多个软引用周期导致重复加载的问题，因为设置后还是会在下一个周期回收软引用，仍然会有跨引用周期和并发生成重复类的问题无法完全避免，而且并发导致重复生成GMA也是仍然存在的。

>最佳实践：默认值1000毫秒，一般可将`SoftRefLRUPolicyMSPerMB`设为1000~10000毫秒

### 4.1.5 升级高版本JDK

高版本JDK有一些反射针对性的优化，可以参考4.3节

## 4.2 应用实战调优

4.1节提供了几种不同的JVM调优方案，笔者在应用中也有实践过，综合考虑实现成本以及B端应用对反射JNI调用额外增加的耗时（小于1ms）是可以接受的 ，实际优化方案是采取在JVM参数中增加 `-Dsun.reflect.inflationThreshold=2147483647` 参数来彻底关闭inflation机制，调整后效果如图32
![image](/img/20231222-32.jpg)
<center style="color:gray">图32 应用调优后的类加载情况</center>

可以明显地观察到调整后类加载的速度放缓了，笔者的应用是对RT不太敏感的，如果对RT敏感的应用需要适当地调大元空间大小和`inflationThreshold`的值。举个例子，比如笔者的应用也可以调整为

```shell
-XX:MaxMetaspaceSize=512m  
-XX:SoftRefLRUPolicyMSPerMB=1000  
-Dsun.reflect.inflationThreshold=1000
```

这样调整后元空间更大、软引用回收更慢、inflation阈值更大，那么生成GMA类会放缓，虽然不能完全避免，但能够在反射调用耗时和类加载增长中找到一个平衡

## 4.3 其他版本的JDK是否有类似的问题

本文是聚焦在JDK 8的实现，对于其他版本的JDK是否也存在类似的问题，以及要如何去优化呢？

### JDK 7中的反射类相关加载

在JDK 7中虽然没有元空间，但因为永久代也存在类似类加载的问题，所以如果有碰到类似的类频繁加载永久代溢出的问题也可以参考上面的参数调优方案
![image](/img/20231222-33.jpg)
<center style="color:gray">图33</center>

### JDK 11中的反射类相关加载

在JDK11中仍然存在并发安全和软引用回收的问题，`NativeMethodAccessorImpl`的`invoke()`方法中，判定inflation机制的地方仍然未加同步机制，在并发环境下仍会重复生成GMA类
![image](/img/20231222-34.jpg)
<center style="color:gray">图34</center>

### JDK 17中的反射类相关加载

在JDK17中通过CAS来避免多线程重复生成，可以看到在标红1处判断是否已生成GMA的标记`generated==0`，然后紧接着使用CAS将generated置为1，那么后面的反射调用执行到标红1处自然判定为false就不会进入GMA类的生成代码里了，CAS解决了并发线程安全的问题，在并发环境下最多生成一个GMA类，但仍然没有解决软引用周期的问题。如果读者是使用的JDK 17版本碰到重复加载的问题，可以考虑从软引用周期入手，通过调整`SoftRefLRUPolicyMSPerMB`来控制软引用回收频率，也可以适当调大-`Dsun.reflect.inflationThreshold`来降低生成GMA的频率
![image](/img/20231222-35.jpg)
<center style="color:gray">图35</center>

### JDK 21中的反射类相关加载

在JDK21中新引入了`DirectMethodHandleAccessor`也是`MethodAccessor`的一种实现，在前文提到`sun.reflect.ReflectionFactory#newMethodAccessor`方法生成`MethodAccessor`的地方（图36）可以看到，通过`jdk.internal.reflect.ReflectionFactory#useMethodHandleAccessor`来判断是否使用新的实现类（默认实现类，详见图37），同时也兼容了低版本的NativeMethodAccessorImpl实现，可以通过参数`-Djdk.reflect.useDirectMethodHandle`来控制默认实现类方式（图37标红4），如果置为false就仍会走到`NativeMethodAccessorImpl`的实现中`DirectMethodHandleAccessor`默认实现就是JNI调用（图38），从根本上避免了GMA类的生成。
![image](/img/20231222-36.jpg)
<center style="color:gray">图36</center>
![image](/img/20231222-37.jpg)
<center style="color:gray">图37</center>
![image](/img/20231222-38.jpg)
<center style="color:gray">图38</center>

# 5 元空间OOM的其他典型案例分享

元空间OOM的排查相对来说套路比较固定，一般来说就是:1）检查JVM参数配置，查看JVM元空间大小分配；2）增加参数排查类加载卸载信息；3）反编译类定位业务场景，结合业务或框架代码定位根因。除了上文反射的问题外，笔者的团队也出过类似的其他问题，我们来看看具体的例子。

## 5.1 关于元空间大小的设置

笔者团队的一个应用从JDK 7升级到JDK 8后，每次JVM重启都会有两次FullGC会触发告警，虽然不影响后续对外提供服务，但每次重启必现告警也确实十分恼人。

可以通过java命令`（java -XX:+PrintFlagsFinal -version | grep  Meta）`查看java进程的元空间大小发现元空间大小不到21MB，继续检查JVM参数发现是因为升级JDK 8后没有指定`-XX:MetaspaceSize`和`-XX:MaxMetaspaceSize`参数，所以元空间默认大小就是不到21MB，在JVM启动过程中加载类会进行元空间扩容，元空间扩容前会触发`Full GC`，因为有两次扩容所以会触发两次`Full GC`。

```shell
[localhost ~]$ java -XX:+PrintFlagsFinal -version | grep  Meta
    uintx InitialBootClassLoaderMetaspaceSize       = 4194304                             {product}
    uintx MaxMetaspaceExpansion                     = 5451776                             {product}
    uintx MaxMetaspaceFreeRatio                     = 70                                  {product}
    uintx MaxMetaspaceSize                          = 18446744073709547520                    {product}
    uintx MetaspaceSize                             = 21807104                            {pd product}
    uintx MinMetaspaceExpansion                     = 339968                              {product}
    uintx MinMetaspaceFreeRatio                     = 40                                  {product}
     bool TraceMetadataHumongousAllocation          = false                               {product}
     bool UseLargePagesInMetaspace                  = false                               {product}
java version "1.8.0_45"
Java(TM) SE Runtime Environment (build 1.8.0_45-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.45-b02, mixed mode)
```

```shell
-XX:PermSize=384m
-XX:MaxPermSize=384m
```

而修改后加上相应的参数后再重启就没有Full GC告警了。

```shell
-XX:MetaspaceSize=384m
-XX:MaxMetaspaceSize=384m
```

## 5.2 持续类加载导致元空间OOM的另一种场景

从 JDK 8 开始，`Nashorn`取代`Rhino(JDK 6, JDK 7) `成为 Java 的嵌入式 JavaScript 引擎。Nashorn 完全支持 ECMAScript 5.1 规范以及一些扩展。它使用基于 JSR 292 的新语言特性，其中包含在 JDK 7 中引入的 invokedynamic，将 JavaScript 编译成 Java 字节码。与先前的 Rhino 实现相比，这带来了2到10倍的性能提升。

笔者团队的另一个项目中使用了Nashorn包的JavaScript脚本执行能力做数据监控的规则判定，线上每隔一段时间就会触发元空间OOM告警，需要手动重启才能恢复。

有了我们之前的经验，我们在分析时直接看类加载的日志统计加载最多的类，然后结合业务代码和源码逐步断点排查，基本就能定位到根因了。在排查类加载中发现是有大量的类被加载。

`[Loaded jdk.nashorn.internal.scripts.Script$1474$\^eval\_ from __JVM_DefineClass__]`

这里大概解释一下现象：Nashorn引擎本身会判断表达式是否已经编译过，如果没有编译过就会重新编译一次表达式，编译时会加载一个Script的类，判断表达式是否编译过是根据表达式的字符串来匹配的，所以当表达式不一样时就会重新编译表达式。以为表达式用到了大量的变量替换所以替换后的表达式大多都不一致，会重复加载Script类最终导致MetaSpace占满，触发频繁的Full GC。
![image](/img/20231222-39.jpg)
<center style="color:gray">图39 表达式相关Script类的生成</center>

举个例子，表达式为：`${BrandId}>0 || ${shopId}>0`那么当门店shopId不同时，表达式就可能为：`${12345}>0 || ${222222}>0 或 ${12345}>0 || ${333333}>0`，那么这两个就是不同的表达式，会被编译两次加载两个Script类，所以框架每次执行新请求时都可能会生成新的类。最终导致元空间内存溢出OOM。

查阅了文档后发现Nashorn还有一种参数绑定的调用形式，可以只编译原始表达式（1次）通过把动态参数传入框架做匹配，这样同一个表达式就不会被编译多次，频繁类加载问题也迎刃而解了。

最佳实践：如果应用中有用到动态类加载的方式，如表达式执行引擎，groovy脚本，反射等，需要特别关注下类加载的情况，避免持续类加载的情况。

# 6  总结

## 6.1 方法区大小问题

JDK 8之前的永久代也可能会有类似的永久代OOM的情况，需要注意永久代的大小，排查的方法和JDK 8也基本类似的。

JDK 8的元空间的默认初始大小只有20.75MB（64位JVM），注意元空间的大小设置避免元空间扩容和空间不足。

## 6.2 相关优化参数

|参数|说明|备注|
| ----------- | ----------- | ----------- |
|inflationThreshold参数|为避免反射膨胀可以适当调整|注意Oracle JVM和IBM JVM对于0的实现不同[^3]|
|noInflation参数|反射字节码优化开关|默认值=false|
|SoftRefLRUPolicyMSPerMB参数|软引用回收LRU策略关键参数|默认值=0|

## 6.3 监控层面

增加类加载和元空间大小相关的配置项，在OOM前提前预警问题。

配置应用主动GC释放内存空间。

## 6.4 代码层面

尽量避免反射的使用，性能不好同时还可能有类加载的问题，可以尝试用hession、protobuf等高性能序列化方式，如无法避免请注意前面提到的相关优化参数。

尽量避免Groovy脚本，表达式执行引擎等重复加载类的情况，如有必要的话优化使用方式。

## 6.5 研究方向

目前网上关于JDK 21的资料比较少，后续继续研究下JDK 21下`DirectMethodHandleAccessor`的实现，看下在默认JNI调用下如何实现更好的性能，对这部分感兴趣的同学可以参考：[method handles](https://blogs.oracle.com/javamagazine/post/java-reflection-method-handles)

# 7 参考文献
1.[R大-关于反射调用方法的一个log](https://blog.csdn.net/rednaxelafx/article/details/83517409)

2.《深入理解Java虚拟机》-周志明

3.[OpenJDK](http://openjdk.java.net/jeps/122)

4.[ASM](http://attic-distfiles.pld-linux.org/distfiles/distfiles/by-md5/5/f/5f17bfac3563feb108793575f74ce27c/asm-eng.pdf)

[^1]:inflation机制：参考第3.2.1节
[^2]:GeneratedMethodAccessor：这个是字节码生成的方法描述类，源码中是找不到这个类的，标在这里是为了让大家知道除了native实现外还有一种GMA字节码的方式
[^3]:注意Oracle JVM和IBM JVM对于0的实现不同：Oracle JVM的实现里0是指不设阈值每次都会生成字节码类，而IBM JVM实现里0是关闭inflation机制
