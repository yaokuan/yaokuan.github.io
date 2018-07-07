---
layout:     post
title:      Java主线程等待子线程的实现方式
date:       2018-07-07 18:02:00
categories: 多线程
tags:       Java 多线程 高并发
---

* 文章目录
{:toc}

最近遇到多线程里面一个常见的问题：“如何让主线程在全部子线程执行完毕后再继续执行？”，然后就整理了几种常见的实现方式

## 方法一：主线程sleep
主线程等待子线程执行完最简单的方式当然是在主线程中Sleep一段时间，这种方式最简单，我们先看下实现

```java
    private static class MyThread extends Thread {
        @Override
        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(String.format("%s %s was finished", DateUtils.format(new Date(), "hh:mm:ss:SSS"), getName()));
        }
    }

    public static void main(String[] args) {
        System.out.println(String.format("%s I was main and I'm started", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
        for (int i = 0; i < 10; i++) {
            MyThread myThread = new MyThread();
            myThread.start();
        }
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(String.format("%s I was main and I'm finished", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
    }
```
但这种方式有个非常大的弊端：无法准确地预估全部子线程执行完毕的时间。时间太久，主线程就需要空等；时间太短，子线程又可能没有全部执行完毕。

## 方法二：子线程join
那么另外一种方式就是使用线程的Join()方法来实现主线程的等候，我们还是先来看下实现代码

```java
    //此处省略MyThread代码，同上
    
    public static void main(String[] args) {
        System.out.println(String.format("%s I was main and I'm started", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            MyThread myThread = new MyThread();
            myThread.start();
            threads.add(myThread);
        }
        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println(String.format("%s I was main and I'm finished", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
    }
```
可以看到我们在全部子线程开始执行后，又在主线程中执行全部子线程的join方法，那么主线程会等待全部子线程执行完毕后继续往下执行。这里放上Java 7 Concurrency Cookbook对join方法的定义，个人认为比JDK中定义更准确。

> join() method suspends the execution of the calling thread until the object called finishes its execution.

我们看下join方法的源码

```java
    public final void join() throws InterruptedException {
        join(0);
    }
    
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }

```
可以看到，如果join的参数为0，那么主线程会一直判断自己是否存活，如果主线程存活，则调用主线程的wait()方法，那么我们继续看下wait方法的定义

```java
/**
     * Causes the current thread to wait until either another thread invokes the
     * {@link java.lang.Object#notify()} method or the
     * {@link java.lang.Object#notifyAll()} method for this object, or a
     * specified amount of time has elapsed.
     */
    public final native void wait(long timeout) throws InterruptedException;
```
大概意思是说wait(0)会使主线程进入睡眠状态，直到调用join方法的线程（简称t线程，下同）执行完毕后调用notify()或notifyAll()将其唤醒。注意这个地方有几点需要特别说明：
1. 代码中没有显示调用notify或notifyAll的地方，这个唤醒操作其实是由于Java虚拟机在线程执行完毕后所做的
2. 主线程需要获得t线程的对象锁（wait 意味着拿到该对象的锁），然后进入睡眠状态
3. 由于每个子线程都执行了join方法，所以主线程需要等待全部子线程执行完毕后才能被唤醒

## 方法三：使用CountDownLatch
另外我们还可以使用java.util.concurrent包里的CountDownLatch，初始设置和子线程个数相同的计数器，子线程执行完毕后计数器减1，直到全部子线程执行完毕。注意countDownLatch不可能重新初始化或者修改CountDownLatch对象内部计数器的值，一个线程调用countdown方法happen-before另外一个线程调用await方法
```java
    private static CountDownLatch latch = new CountDownLatch(10);

    public static void main(String[] args) {
        System.out.println(String.format("%s I was main and I'm started", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
        for (int i = 0; i < 10; i++) {
            MyThread myThread = new MyThread();
            myThread.start();
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(String.format("%s I was main and I'm finished", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
    }
```
关于CountDownLatch的具体实现这里不再详细展开，如有兴趣可以[戳这里](https://www.jianshu.com/p/7c7a5df5bda6?ref=myread)，我们目前只需要直到await会使主线程阻塞直到计数器清零即可

## 方法四：使用CycleBarrier

另外还可以使用CycleBarrier实现主线程等待。CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

我们看下实现代码

```java
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(10);

    private static class MyThread extends Thread {
        @Override
        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(String.format("%s %s was finished", DateUtils.format(new Date(), "hh:mm:ss:SSS"), getName()));
            try {
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws BrokenBarrierException {
        System.out.println(String.format("%s I was main and I'm started", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
        for (int i = 0; i < 10; i++) {
            MyThread myThread = new MyThread();
            myThread.start();
        }
        try {
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(String.format("%s I was main and I'm finished", DateUtils.format(new Date(), "hh:mm:ss:SSS")));
    }
```

看完方法三和方法四，有人就要问了：“CountDownLatch和CyclicBarrier都能够实现线程之间的等待，这两种方式有什么区别”。别急，下面就来讲下他们的区别
- CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行，而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；
- CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的(reset)。



