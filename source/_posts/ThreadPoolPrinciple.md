---
title: 从使用到原理学习Java线程池
date: 2018-05-04 14:38:55
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

![](20180319153538.jpg)  

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
如果线程池的大小超过了处理任务所需的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完成依赖于操作系统（或者说JVM）能够创建的最大线程大小。
*ScheduledThreadPool：*核心线程池固定，大小无限的线程池。此线程池支持定时以周期性执行任务的需求。
``` java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```
创建一个周期性执行任务的线程池。如果闲置，非核心线程池会在DEFAULT_KEEPALIVE_MILLIS时间内回收。
线程池最常用的提交任务方法有两种：
*execute：*
``` java
ExecutorService.execute(Runnable runnable);
```
*submit：*
``` java 
FutureTask task=ExecutorService.submit(Runnable runnable);
FutureTask task=ExecutorService.submit(Runnable runnable,T Result);
FutureTask task=ExecutorService.submit(Callable<T> callable);
```
submit(Callable callable)的实现，submit(Runnable runnable)同理。
``` java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    FutureTask<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
可以看出submit开启的是有返回结果的任务，会返回一个FutureTask对像，这样就能通过get()方法得到结果。submit最终调用的也是execute(Runnable runnable),submit只是将Callable对象或Runnable封装成一个FutureFask对象，因为FutureTask是个Runnable，所以可以在execute中执行。关于Callable对象和Runnable怎么封装成FutureTask对象，见[Callable和Future、FutureTask的使用。](http://www.silencedut.com/2016/06/15/Callable%E5%92%8CFuture%E3%80%81FutureTask%E7%9A%84%E4%BD%BF%E7%94%A8/)

**线程池实现原理**
如果只讲线程池的使用，那么招聘博客没有什么大的价值,充其量也就是熟悉Executor相关API的过程。程序池的实现过程没有用到synchronized关键字，用的都是volatile,Lock和同步(阻塞)队列，Atomic相关类，FutureTask等等，因为后者性能更优。理解的过程可以很好的学习源码中并发控制思想。

在开篇提到过线程池的优点是可以总结为一下三点：
* 线程复用
* 控制最大并发数
* 管理线程

1. ***线程复用过程***
    理解线程复用原理首先应了解线程的生命周期。
    ![](006tKfTcjw1f78hxvv1kmj30nm09c75z.jpg)  
    在线程的生命周期中，它要经过新建(New)、就绪(Runnable)、运行(Running)、阻塞(Bloched)和死亡(Dead)5种状态。

    Thread通过new来新建一个线程，这个过程是初始化一些线程信息，如线程名，id，线程所属group等，可以认为只是个普通的对象。调用Thread的start()后Java虚拟机会为其创建调用栈和程序计数器，同时将hasBeenStarted为true，之后调用start方法就会有异常。

    处于这个状态中的线程并没有开始运行，只是表示线程可以运行了。至于该线程何时开始运行，取决于JVM里线程调度器的调度。当线程获取cpu后，run()方法会被调用。不要自己去调用Thread的run()方法。之后根据CPU的调度在(就绪--运行--阻塞)间切换，直到run()方法结束或其他方式停止线程，进入dead状态。

    所以实现线程复用的原理应该就是要保持线程处于存活状态(就绪、运行、阻塞)。接下来看看下ThreadPoolExecutor是怎么实现线程复用的。

    在ThreadPoolExecutor主要Worker类来控制线程的复用。开下Worker类简化后的代码，这样方便理解：

    ``` java
    private final class Worker implements Runnable {
    
    	final Thread thread;

    	Runnable firstTask;

    	Worker(Runnable firstTask) {
    		this.firstTask = firstTask;
    		this.thread = getThreadFactory().newThread(this);
    	}

    	public void run() {
    		runWorker(this);
    	}
    
    	final void runWorker(Worker w) {
    		Runnable task = w.firstTask;
    		w.firstTask = null;
    		while (task != null || (task = getTask()) != null){
    		    task.run();
            }
    	}
    }
    ```
    Worker是一个Runnable，同时拥有一个thread，这个thread就是要开启的线程，在新建Worker对象时同时新建一个Thread对象，同时将Worker自己作为参数传入Thread，这样当Thread的start()方法调用时，运行的实际上是Worker的run()方法，接着到runWorker()中，有个while循环，一直从getTask()里得到Runnable对象，顺序执行。getTask()又是怎么得到Runnable对象得呢？依旧是简化后的代码：
    ``` java
    private Runnable getTask() {
        if(一些特殊情况) {
            return null;
        }

        Runnable r = workQueue.take();

        return r;
    }
    ```
    这个workQueue就是初始化ThreadPoolExecutor时存放的任务的BlockingQueue队列，这个队列里的存放的都时将要执行的Runnable任务。因为BlockingQueue是个阻塞队列，BlockingQueue.task()得到如果是空，则进入等待状态直到BlockingQueue有新的对象被加入时唤醒阻塞的线程。所以一般情况Thread的run()方法就不会结束，而是不断的执行从workQueue里的Runnable任务，这就达到了线程复用的原理了。
2. ***控制最大并发数***
    那Runnable是什么时候放入workQueue?Worker又是什么时候创建，Worker里的Thread的又是什么时候调用start()开启新线程来执行Worker的run()方法的呢？有上面的分析看出Worker里的runWorker()执行任务是一个接一个，串行进行的。那并发是怎么体现的呢？

    很容易想到是在execute(Runnable runnable)时会做上面的一些任务。看下execute里是怎么做的。

    简化后的代码 execute：
    ``` java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

         int c = ctl.get();
        // 当前线程数 < corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            // 直接启动新的线程。
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 活动线程数 >= corePoolSize
        // runState为RUNNING && 队列未满
        // workQueue.offer(command)表示添加到队列，如果添加成功返回true，否则返回   false
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次检验是否为RUNNING状态
            // 非RUNNING状态 则从workQueue中移除任务并拒绝
            if (!isRunning(recheck) && remove(command))
                reject(command);// 采用线程池指定的策略拒绝任务
            // 两种情况：
            // 1.非RUNNING状态拒绝新的任务
            // 2.队列满了启动新的线程失败（workCount > maximumPoolSize）
        } else if (!addWorker(command, false))
            reject(command);
    }
    ```
    简化后的代码 addWorker：
    ``` java
    private boolean addWorker(Runnable firstTask, boolean core) {

        int wc = workerCountOf(c);
        if (wc >= (core ? corePoolSize : maximumPoolSize)) {
            return false;
        }

        w = new Worker(firstTask);
        final Thread t = w.thread;
        t.start();
    }
    ```
    根据代码再看上面提到的线程池工作过程中的添加任务的情况：
        1. 如果正在运行的线程数小于corePoolSize，那么马上创建线程运行这个任务；
        2. 如果正在运行的线程数大于或等于corePoolSize，那么将这个任务放入队列；
        3. 如果这个时候队列满了，而且正在运行的线程数量小于maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；
        4. 如果队列满了，而且正在运行的线程数量大于或者等于maximumPoolSize，那么线程池会抛出异常RejectExecutionException。
    > 这就是Android的AsyncTask在并行执行是在超出最大任务数是抛出RejectExecutionException的原因所在，详见基于最新版本的AsyncTask源码解读及AsyncTask的黑暗面

    通过addWorde如果成功创建新的线程成功，则通过start()开启新的线程，同时将firstTask作为这个Woker里的run()中执行的第一个任务。

    虽然每个Worker的任务都是串行处理的，但是如果创建多个Worker，因为是公用一个workQueue，所以就会并行处理了。

    所以根据corePoolSize和maximumPoolSize来控制最大并发数。大致过程可用下图表示。
    ![](006tKfTcgw1f79ope94y3j30nm0egdif.jpg)
    上面的讲解和图可以很好的理解的这个过程。

    如果是做Android开发的，并且对Handler原理比较熟悉，你可能会觉得这个图挺熟悉，其中的一些过程和Handler，Looper，Meaasge使用中，很相似。Handle。send(Message)相当于execute(Runnable)，Looper中维护的Message队列相当于BlockingQueue，只不过需要自己通过同步来维护这个队列，Looper中的loop()函数循环从Message队列取Message和Worker中的runWork()不断从BlockingQueue取Runnable是同一的道理。

3. ***管理线程***
    通过线程池可以很好的管理线程的复用，控制并发数，以及销毁等过程，线程的复用和控制并发上面已经讲了，而线程的管理过程已经穿插在其中了，也很好理解。

    在ThreadPoolExecutor有个ctl的AtomicInteger变量。通过这一个变量保存了两个内容：
        1. 所有线程的数量
        2. 每个线程所处的状态
    
    其中低29位存线程数，高3位存runState,通过位运算来得到不同的值。

    ``` java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    //得到线程的状态
    private static int runStateOf(int c) {
        return c & ~CAPACITY;
    }

    //得到Worker的的数量
    private static int workerCountOf(int c) {
        return c & CAPACITY;
    }


    // 判断线程是否在运行
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
    ```

    这里主要通过shutdown和shutdownNow()来分析线程池的关闭过程。首先线程池有五种状态来控制任务添加与执行。主要介绍以下三种：
        1. RUNNING状态：线程池正常运行，可以接受新的任务并处理队列中的任务；
        2. SHUTDOWN状态：不在接受新的任务，但是会执行队列中的任务；
        3. STOP状态；不在接受新任务，不处理队列中的任务；
    
    shutdown这个方法会将runState置为SHUNDOWN，会终止所有的空闲线程，而仍在工作的线程不受影响。所以队列中的任务人会被执行。shutdownNow方法runState置为STOP。和shutdown方法的区别，这个方法会终止所有的线程，所以队列中的任务也不会被执行了。

**总结**

通过对ThreadPoolExecutor源码的分析，从总体上了解了线程的创建，任务的添加，执行等过程，熟悉这些过程，使用线程池就会更新松了。
而从中学到的一些对并发控制，以及生产者--消费者模型任务处理器的使用，对以后理解或解决其他相关问题会有很大的帮助。比如Android中的Handler机制，而Looper中的Messager队列用一个BlookQueue来处理同样是可以的，这些就是读源码的收获吧