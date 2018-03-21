---
title: 从使用到原理学习Java线程池
date: 2018-03-19 14：38：55
tags: 笔记
---
**线程池技术背景**
在面向对象编程中，创建和销毁对象都是很费时间的，因为创建一个对象要获取内存资源或者其他更多资源。在Java中更是如此，虚拟机将试图跟踪每一个对象，以便能够在对象销毁后进行垃圾回收。
<!-- more -->
所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数，特别是一些很耗资源的对象创建和销毁。如何利用已有对象来服务就是一个需要解决的关键问题，其实这就是一些“池化资源”技术产生的原因。

例如Android中常见到的很多通用组件一般都离不开“池”的概念，如各种图片加载库，网络请求库，即使Android的消息传递机制的Message当使用Message.obtain()就是使用Message池中的对象，因为此这个概念很重要。本文将介绍的线程池技术同样符合这一思想。

**线程池的优点：**
1. 重用线程池中的线程，减少因对象创建、销毁所带来的性能开销；
2. 能有效的控制线程的最大并发数，提高系统资源利用率，同时避免过多的资源竞争，避免堵塞；
3. 能够线程进行简单的管理，使线程的使用简单、高效；

线程池框架Executor
Java中的线程池是通过Executor框架实现的，Executor框架包括类：Executor、Executors、ExecutorService、ThreadPoolExecutor、Callable和Future、FutureTask的使用等。

![alt](20180319153538.jpg)  

**Executor：**所有线程池的接口，只有一个方法。
``` java
public interface Executor{
    void execute(Runnable command);
}
```
**ExecutorService：**增加Executor的行为，是Executor实现类最直接接口。
**Executors：**提供了一系列工厂方法用于创建线程池，返回的线程池都实现了ExecutorService接口。
``` java 
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue) {
    this(corePoolSize,maximumPoolSize,keepAliveTime,workQueue,Executors.defaultThreadFactory(),defaultHandler);
}
```
*corePoolSize：*线程池的核心线程数，线程池中运行的线程数也永远不会超过corePoolSize个，默认情况下可以一直存活。可以通过设置allowCoreThreadTimeOut为true，此时核心线程数就是0，此时keepAliveTime控制所有线程的超时时间；
*maximumPoolSize：*线程池运行的最大线程数；
*keepAliveTime：*指的是空闲线程结束的超时时间；
*unit：*是一个枚举，表示keepAliveTime的单位；
*workQueue：*表示存放任务的BlockingQueue<Runnable>队列。
> BlockingQueue：阻塞队列是java.util.concurrent下的主要控制线程同步的工具类，如果BlockingQueue是空的，从BlockingQueue取东西的操作将会被阻断进入等待状态，直到BlockingQueue进了东西才会被唤醒，同样，如果Blocking是满的，任何试图往里存东西的操作也会被阻断进入等待状态，直到BlockingQueue里有空间会被唤醒继续操作。
阻塞队列常用于生产者和消费者的场景，生产者就是往队列里添加元素的线程，消费者就是从队列里拿元素的线程。具体的实现类有LinkedBlockingQueue，ArrayBlockingQueued等。一般其内部的都是Lock和Condition来实现阻塞和唤醒。

线程池的工作过程如下：
线程池刚创建时，里面没有一线程。任务队列是作为参数传进来的。不过，就算队列里面有任务，线程池也不会马上执行它们。
当调用execute()方法添加一个任务时，线程池会做如下判断：
>如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行这个任务；
如果正在运行的线程数大于或等于corePoolSize，那么将这个任务放入队列；
如果这个时候队列满了，而且正在运行的线程数量小于maximumPoolSize，那么还是要创建非核心线程运行这个任务；
如果队列满了，而且正在运行的线程数量大于或等于maximumPoolSize，那么线程池抛出异常RejectExecutionException。
当一个线程完成任务时，它会从队列中取下一个任务和来执行。
当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于corePoolSize,那么这个线程就停掉。所以线程池的所有任务完成后，它最终会收缩到corePoolSize的大小。

**线程的创建和使用**
生成线程池采用工具类Executors的静态方法，以下是几种常见的线程池。
*SingleThreadExecutor：*单个后台线程（其缓冲队列是无界的）
``` java
public static ExecutorService newSingleThreadExecutor(){
    return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```
创建一个单线程的线程池。这个线程池只有一个核心线程工作，也就相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有的任务的执行顺序按照任务的提交顺序执行。
*FixedThreadPool：*只有核心线程的线程池，大小固定（其缓冲队列是无界的）
``` java
public static ExecutorSerice newFixedThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```
创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新的线程。
*CachedThreadPool：*无界线程池，可以进行自动线程回收。
``` java 
public static ExecutorService newCachedThreadPool(){
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```
