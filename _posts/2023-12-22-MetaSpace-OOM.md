---
layout:     post
title:      深入反射解密JVM元空间OOM
date:       2023-12-22 22:24:00
categories: JVM
tags:       Java GC JVM
---

* 文章目录
{:toc}


# 背景
笔者在生产实践中碰到过很多OOM相关的问题，其中因为元空间引发的OOM问题不在少数，大多数堆内存的问题结合JVM参数、GC日志、堆dump分析还是比较容易有清晰的方向的，而元空间的问题往往是比较难定位的。因此笔者借助一次元空间OOM的排查经历来详细了解下什么是元空间，为什么元空间会OOM以及如何避免。为了让读者能够快速有个大概的印象，先介绍下本文的写作思路，本文会从问题描述，介绍元空间，分析案例现象引出频繁类加载问题，在过程中会了解反射执行过程，初识GeneratedMethodAccessor类，分析GeneratedMethodAccessor类重复加载的原因，进行实验验证，JVM参数调优实践。这个过程中会涉及到反射调用的实现细节，膨胀（inflation）机制及其解决的问题，反射和软引用的关系，反射过程中的线程安全，JVM参数调优等知识点，本文主要围绕JDK 8来展开排查和调优过程，也会介绍JDK 7/11/17/21等版本的反射实现和调优思路。

# 一个线上元空间OOM案例
## 简述线上元空间OOM现象
线上某个应用突然开始不停地有 java.lang.OutOfMemoryError: Metaspace 告警，看元空间占用内存一直在增长的，一般来说元空间的增长都是因为类加载导致的。继续看了下应用的 jvm.classloading.loaded.count 指标趋势图，基本一直在增长直到下一次服务发布或者是Full GC时才会降下来一些，如果应用间隔了一段时间没有发布就会频繁Full GC直到元空间OOM告警，为了搞清楚元空间OOM的根因，首先得了解下什么是元空间。

## 附上Java GC CheatSheet
![image](http://oc26wuqdw.bkt.clouddn.com/2018/8/jvm/Java%207%20-%20GC%20cheatsheet.png)
