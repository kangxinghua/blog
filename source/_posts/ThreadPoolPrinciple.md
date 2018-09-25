---
title: 从使用到原理学习Java线程池
date: 2018-05-04 14:38:55
tags: 笔记
---
## 线程池技术背景
在面向对象编程中,创建和销毁对象都是很费时间的,因为创建一个对象要获取内存资源或者其他更多资源.在Java中更是如此,虚拟机将试图跟踪每一个对象,以便能够在对象销毁后进行垃圾回收.
<!-- more -->
所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数,特别是一些很耗资源的对象创建和销毁.如何利用已有对象来服务就是一个需要解决的关键问题,其实这就是一些“池化资源”技术产生的原因.

例如Android中常见到的很多通用组件一般都离不开“池”的概念,如各种图片加载库,网络请求库,即使Android的消息传递机制的Message当使用Message.obtain()就是使用Message池中的对象,因为此这个概念很重要.本文将介绍的线程池技术同样符合这一思想.

## 线程池的优点
1. 重用线程池中的线程,减少因对象创建、销毁所带来的性能开销；
2. 能有效的控制线程的最大并发数,提高系统资源利用率,同时避免过多的资源竞争,避免堵塞；
3. 能够线程进行简单的管理,使线程的使用简单、高效；

线程池框架Executor
Java中的线程池是通过Executor框架实现的,Executor框架包括类：Executor、Executors、ExecutorService、ThreadPoolExecutor、Callable和Future、FutureTask的使用等.

![](20180319153538.jpg)  

**Executor：**所有线程池的接口,只有一个方法.
``` java
public interface Executor{
    void execute(Runnable command);
}
```
**ExecutorService：**增加Executor的行为,是Executor实现类最直接接口.
**Executors：**提供了一系列工厂方法用于创建线程池,返回的线程池都实现了ExecutorService接口.
``` java 
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue) {
    this(corePoolSize,maximumPoolSize,keepAliveTime,workQueue,Executors.defaultThreadFactory(),defaultHandler);
}
```
*corePoolSize：*线程池的核心线程数,线程池中运行的线程数也永远不会超过corePoolSize个,默认情况下可以一直存活.可以通过设置allowCoreThreadTimeOut为true,此时核心线程数就是0,此时keepAliveTime控制所有线程的超时时间；
*maximumPoolSize：*线程池运行的最大线程数；
*keepAliveTime：*指的是空闲线程结束的超时时间；
*unit：*是一个枚举,表示keepAliveTime的单位；
*workQueue：*表示存放任务的BlockingQueue<Runnable>队列.
> BlockingQueue：阻塞队列是java.util.concurrent下的主要控制线程同步的工具类,如果BlockingQueue是空的,从BlockingQueue取东西的操作将会被阻断进入等待状态,直到BlockingQueue进了东西才会被唤醒,同样,如果Blocking是满的,任何试图往里存东西的操作也会被阻断进入等待状态,直到BlockingQueue里有空间会被唤醒继续操作.
阻塞队列常用于生产者和消费者的场景,生产者就是往队列里添加元素的线程,消费者就是从队列里拿元素的线程.具体的实现类有LinkedBlockingQueue,ArrayBlockingQueued等.一般其内部的都是Lock和Condition来实现阻塞和唤醒.

线程池的工作过程如下：
线程池刚创建时,里面没有一线程.任务队列是作为参数传进来的.不过,就算队列里面有任务,线程池也不会马上执行它们.
当调用execute()方法添加一个任务时,线程池会做如下判断：
>如果正在运行的线程数量小于corePoolSize,那么马上创建线程运行这个任务；
如果正在运行的线程数大于或等于corePoolSize,那么将这个任务放入队列；
如果这个时候队列满了,而且正在运行的线程数量小于maximumPoolSize,那么还是要创建非核心线程运行这个任务；
如果队列满了,而且正在运行的线程数量大于或等于maximumPoolSize,那么线程池抛出异常RejectExecutionException.
当一个线程完成任务时,它会从队列中取下一个任务和来执行.
当一个线程无事可做,超过一定的时间（keepAliveTime）时,线程池会判断,如果当前运行的线程数大于corePoolSize,那么这个线程就停掉.所以线程池的所有任务完成后,它最终会收缩到corePoolSize的大小.

## 线程的创建和使用
生成线程池采用工具类Executors的静态方法,以下是几种常见的线程池.
*SingleThreadExecutor：*单个后台线程（其缓冲队列是无界的）
``` java
public static ExecutorService newSingleThreadExecutor(){
    return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```
创建一个单线程的线程池.这个线程池只有一个核心线程工作,也就相当于单线程串行执行所有任务.如果这个唯一的线程因为异常结束,那么会有一个新的线程来替代它.此线程池保证所有的任务的执行顺序按照任务的提交顺序执行.
*FixedThreadPool：*只有核心线程的线程池,大小固定（其缓冲队列是无界的）
``` java
public static ExecutorSerice newFixedThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```
创建固定大小的线程池.每次提交一个任务就创建一个线程,直到线程达到线程池的最大大小.线程的大小一旦达到最大值就会保持不变,如果某个线程因为执行异常而结束,那么线程池会补充一个新的线程.
*CachedThreadPool：*无界线程池,可以进行自动线程回收.
``` java 
public static ExecutorService newCachedThreadPool(){
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```
如果线程池的大小超过了处理任务所需的线程,那么就会回收部分空闲（60秒不执行任务）的线程,当任务数增加时,此线程池又可以智能的添加新线程来处理任务.此线程池不会对线程池大小做限制,线程池大小完成依赖于操作系统（或者说JVM）能够创建的最大线程大小.
*ScheduledThreadPool：*核心线程池固定,大小无限的线程池.此线程池支持定时以周期性执行任务的需求.
``` java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```
创建一个周期性执行任务的线程池.如果闲置,非核心线程池会在DEFAULT_KEEPALIVE_MILLIS时间内回收.
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
submit(Callable callable)的实现,submit(Runnable runnable)同理.
``` java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    FutureTask<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
可以看出submit开启的是有返回结果的任务,会返回一个FutureTask对像,这样就能通过get()方法得到结果.submit最终调用的也是execute(Runnable runnable),submit只是将Callable对象或Runnable封装成一个FutureFask对象,因为FutureTask是个Runnable,所以可以在execute中执行.关于Callable对象和Runnable怎么封装成FutureTask对象,见[Callable和Future、FutureTask的使用.](http://www.silencedut.com/2016/06/15/Callable%E5%92%8CFuture%E3%80%81FutureTask%E7%9A%84%E4%BD%BF%E7%94%A8/)

## 线程池实现原理
如果只讲线程池的使用,那么招聘博客没有什么大的价值,充其量也就是熟悉Executor相关API的过程.程序池的实现过程没有用到synchronized关键字,用的都是volatile,Lock和同步(阻塞)队列,Atomic相关类,FutureTask等等,因为后者性能更优.理解的过程可以很好的学习源码中并发控制思想.

在开篇提到过线程池的优点是可以总结为一下三点：
* 线程复用
* 控制最大并发数
* 管理线程

1. ***线程复用过程***
    理解线程复用原理首先应了解线程的生命周期.
    ![](006tKfTcjw1f78hxvv1kmj30nm09c75z.jpg)  
    在线程的生命周期中,它要经过新建(New)、就绪(Runnable)、运行(Running)、阻塞(Bloched)和死亡(Dead)5种状态.

    Thread通过new来新建一个线程,这个过程是初始化一些线程信息,如线程名,id,线程所属group等,可以认为只是个普通的对象.调用Thread的start()后Java虚拟机会为其创建调用栈和程序计数器,同时将hasBeenStarted为true,之后调用start方法就会有异常.

    处于这个状态中的线程并没有开始运行,只是表示线程可以运行了.至于该线程何时开始运行,取决于JVM里线程调度器的调度.当线程获取cpu后,run()方法会被调用.不要自己去调用Thread的run()方法.之后根据CPU的调度在(就绪--运行--阻塞)间切换,直到run()方法结束或其他方式停止线程,进入dead状态.

    所以实现线程复用的原理应该就是要保持线程处于存活状态(就绪、运行、阻塞).接下来看看下ThreadPoolExecutor是怎么实现线程复用的.

    在ThreadPoolExecutor主要Worker类来控制线程的复用.开下Worker类简化后的代码,这样方便理解：

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
    Worker是一个Runnable,同时拥有一个thread,这个thread就是要开启的线程,在新建Worker对象时同时新建一个Thread对象,同时将Worker自己作为参数传入Thread,这样当Thread的start()方法调用时,运行的实际上是Worker的run()方法,接着到runWorker()中,有个while循环,一直从getTask()里得到Runnable对象,顺序执行.getTask()又是怎么得到Runnable对象得呢？依旧是简化后的代码：
    ``` java
    private Runnable getTask() {
        if(一些特殊情况) {
            return null;
        }

        Runnable r = workQueue.take();

        return r;
    }
    ```
    这个workQueue就是初始化ThreadPoolExecutor时存放的任务的BlockingQueue队列,这个队列里的存放的都时将要执行的Runnable任务.因为BlockingQueue是个阻塞队列,BlockingQueue.task()得到如果是空,则进入等待状态直到BlockingQueue有新的对象被加入时唤醒阻塞的线程.所以一般情况Thread的run()方法就不会结束,而是不断的执行从workQueue里的Runnable任务,这就达到了线程复用的原理了.
2. ***控制最大并发数***
    那Runnable是什么时候放入workQueue?Worker又是什么时候创建,Worker里的Thread的又是什么时候调用start()开启新线程来执行Worker的run()方法的呢？有上面的分析看出Worker里的runWorker()执行任务是一个接一个,串行进行的.那并发是怎么体现的呢？

    很容易想到是在execute(Runnable runnable)时会做上面的一些任务.看下execute里是怎么做的.

    简化后的代码 execute：
    ``` java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

         int c = ctl.get();
        // 当前线程数 < corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            // 直接启动新的线程.
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 活动线程数 >= corePoolSize
        // runState为RUNNING && 队列未满
        // workQueue.offer(command)表示添加到队列,如果添加成功返回true,否则返回   false
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
        1. 如果正在运行的线程数小于corePoolSize,那么马上创建线程运行这个任务；
        2. 如果正在运行的线程数大于或等于corePoolSize,那么将这个任务放入队列；
        3. 如果这个时候队列满了,而且正在运行的线程数量小于maximumPoolSize,那么还是要创建非核心线程立刻运行这个任务；
        4. 如果队列满了,而且正在运行的线程数量大于或者等于maximumPoolSize,那么线程池会抛出异常RejectExecutionException.
    > 这就是Android的AsyncTask在并行执行是在超出最大任务数是抛出RejectExecutionException的原因所在,详见基于最新版本的AsyncTask源码解读及AsyncTask的黑暗面

    通过addWorde如果成功创建新的线程成功,则通过start()开启新的线程,同时将firstTask作为这个Woker里的run()中执行的第一个任务.

    虽然每个Worker的任务都是串行处理的,但是如果创建多个Worker,因为是公用一个workQueue,所以就会并行处理了.

    所以根据corePoolSize和maximumPoolSize来控制最大并发数.大致过程可用下图表示.
    ![](006tKfTcgw1f79ope94y3j30nm0egdif.jpg)
    上面的讲解和图可以很好的理解的这个过程.

    如果是做Android开发的,并且对Handler原理比较熟悉,你可能会觉得这个图挺熟悉,其中的一些过程和Handler,Looper,Meaasge使用中,很相似.Handle.send(Message)相当于execute(Runnable),Looper中维护的Message队列相当于BlockingQueue,只不过需要自己通过同步来维护这个队列,Looper中的loop()函数循环从Message队列取Message和Worker中的runWork()不断从BlockingQueue取Runnable是同一的道理.

3. ***管理线程***
    通过线程池可以很好的管理线程的复用,控制并发数,以及销毁等过程,线程的复用和控制并发上面已经讲了,而线程的管理过程已经穿插在其中了,也很好理解.

    在ThreadPoolExecutor有个ctl的AtomicInteger变量.通过这一个变量保存了两个内容：
        1. 所有线程的数量
        2. 每个线程所处的状态
    
    其中低29位存线程数,高3位存runState,通过位运算来得到不同的值.

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

    这里主要通过shutdown和shutdownNow()来分析线程池的关闭过程.首先线程池有五种状态来控制任务添加与执行.主要介绍以下三种：
        1. RUNNING状态：线程池正常运行,可以接受新的任务并处理队列中的任务；
        2. SHUTDOWN状态：不在接受新的任务,但是会执行队列中的任务；
        3. STOP状态；不在接受新任务,不处理队列中的任务；
    
    shutdown这个方法会将runState置为SHUNDOWN,会终止所有的空闲线程,而仍在工作的线程不受影响.所以队列中的任务人会被执行.shutdownNow方法runState置为STOP.和shutdown方法的区别,这个方法会终止所有的线程,所以队列中的任务也不会被执行了.


## 如何正确使用线程池
### 避免使用无界队列
不要使用Executors.newXXXThreadPool()快捷方法创建线程池,因为这种方式会使用无界的任务队列,为避免OOM,我们应该使用ThreadPoolExecutor的构造方法手动指定队列的最大长度：
``` java
ExecutorService executorService = new ThreadPoolExecutor(2, 2, 
                0, TimeUnit.SECONDS, 
                new ArrayBlockingQueue<>(512), // 使用有界队列,避免OOM
                new ThreadPoolExecutor.DiscardPolicy());
```
### 明确拒绝任务时的行为
任务队列总有占满的时候,这是再submit()提交新的任务会怎么样呢？RejectedExecutionHandler接口为我们提供了控制方式,接口定义如下:
``` java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

![](1521946099154-17b29e18-6853-4b39-8e2a-007ea89387bd.png)

拒绝策略 | 拒绝行为
------------ | -------------
AbortPolicy | 抛出RejectedExecutionException
DiscardPolicy | 什么也不做,直接忽略
DiscardOldestPolicy | 丢弃执行队列中最老的任务,尝试为当前提交的任务腾出位置
CallerRunsPolicy | 直接由提交任务者执行这个任务

线程池默认的拒绝行为是AbortPolicy,也就是抛出RejectedExecutionHandler异常,该异常是非受检异常,很容易忘记捕获.如果不关心任务被拒绝的事件,可以将拒绝策略设置成DiscardPolicy,这样多余的任务会悄悄的被忽略.

``` java
ExecutorService executorService = new ThreadPoolExecutor(2, 2, 
                0, TimeUnit.SECONDS, 
                new ArrayBlockingQueue<>(512), 
                new ThreadPoolExecutor.DiscardPolicy());// 指定拒绝策略
```

### 获取处理结果和异常
线程池的处理结果、以及处理过程中的异常都被包装到Future中,并在调用Future.get()方法时获取,执行过程中的异常会被包装成ExecutionException,submit()方法本身不会传递结果和任务执行过程中的异常.获取执行结果的代码可以这样写：

``` java
ExecutorService executorService = Executors.newFixedThreadPool(4);
Future<Object> future = executorService.submit(new Callable<Object>() {
        @Override
        public Object call() throws Exception {
            throw new RuntimeException("exception in call~");// 该异常会在调用Future.get()时传递给调用者
        }
    });
     
try {
  Object result = future.get();
} catch (InterruptedException e) {
  // interrupt
} catch (ExecutionException e) {
  // exception in Callable.call()
  e.printStackTrace();
}
```

## 线程池的常用场景

### 正确构造线程池

``` java
int poolSize = Runtime.getRuntime().availableProcessors() * 2;
BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(512);
RejectedExecutionHandler policy = new ThreadPoolExecutor.DiscardPolicy();
executorService = new ThreadPoolExecutor(poolSize, poolSize,
    0, TimeUnit.SECONDS,
            queue,
            policy);
```

### 获取单个结果
过submit()向线程池提交任务后会返回一个Future,调用V Future.get()方法能够阻塞等待执行结果,V get(long timeout, TimeUnit unit)方法可以指定等待的超时时间.

### 获取多个结果
如果向线程池提交了多个任务,要获取这些任务的执行结果,可以依次调用Future.get()获得.但对于这种场景,我们更应该使用ExecutorCompletionService,该类的take()方法总是阻塞等待某一个任务完成,然后返回该任务的Future对象.向CompletionService批量提交任务后,只需调用相同次数的CompletionService.take()方法,就能获取所有任务的执行结果,获取顺序是任意的,取决于任务的完成顺序：

``` java
void solve(Executor executor, Collection<Callable<Result>> solvers)
   throws InterruptedException, ExecutionException {
    
   CompletionService<Result> ecs = new ExecutorCompletionService<Result>(executor);// 构造器
    
   for (Callable<Result> s : solvers)// 提交所有任务
       ecs.submit(s);
        
   int n = solvers.size();
   for (int i = 0; i < n; ++i) {// 获取每一个完成的任务
       Result r = ecs.take().get();
       if (r != null)
           use(r);
   }
}
```

### 单个任务的超时时间
V Future.get(long timeout, TimeUnit unit)方法可以指定等待的超时时间,超时未完成会抛出TimeoutException.

多个任务的超时时间
等待多个任务完成,并设置最大等待时间,可以通过CountDownLatch完成：
``` java
public void testLatch(ExecutorService executorService, List<Runnable> tasks) 
    throws InterruptedException{
       
    CountDownLatch latch = new CountDownLatch(tasks.size());
      for(Runnable r : tasks){
          executorService.submit(new Runnable() {
              @Override
              public void run() {
                  try{
                      r.run();
                  }finally {
                      latch.countDown();// countDown
                  }
              }
          });
      }
      latch.await(10, TimeUnit.SECONDS); // 指定超时时间
  }
```

## 线程池和装修公司
以运营一家装修公司做个比喻.公司在办公地点等待客户来提交装修请求；公司有固定数量的正式工以维持运转；旺季业务较多时,新来的客户请求会被排期,比如接单后告诉用户一个月后才能开始装修；当排期太多时,为避免用户等太久,公司会通过某些渠道（比如人才市场、熟人介绍等）雇佣一些临时工（注意,招聘临时工是在排期排满之后）；如果临时工也忙不过来,公司将决定不再接收新的客户,直接拒单.

线程池就是程序中的“装修公司”,代劳各种脏活累活.上面的过程对应到线程池上：
``` java
// Java线程池的完整构造函数
public ThreadPoolExecutor(
  int corePoolSize, // 正式工数量
  int maximumPoolSize, // 工人数量上限,包括正式工和临时工
  long keepAliveTime, TimeUnit unit, // 临时工游手好闲的最长时间,超过这个时间将被解雇
  BlockingQueue<Runnable> workQueue, // 排期队列
  ThreadFactory threadFactory, // 招人渠道
  RejectedExecutionHandler handler // 拒单方式
  ) 
```

## 如何合理地估算线程池大小?

这个问题虽然看起来很小,却并不那么容易回答.大家如果有更好的方法欢迎赐教,先来一个天真的估算方法：假设要求一个系统的TPS（Transaction Per Second或者Task Per Second,tps是每秒内的事务数,比如执行了dml操作,那么相应的tps会增加）至少为20,然后假设每个Transaction由一个线程完成,继续假设平均每个线程处理一个Transaction的时间为4s.那么问题转化为：

### 如何设计线程池大小,使得可以在1s内处理完20个Transaction?

计算过程很简单,每个线程的处理能力为0.25TPS(1s/4s),那么要达到20TPS,显然需要20/0.25=80个线程.

很显然这个估算方法很天真,因为它没有考虑到CPU数目.一般服务器的CPU核数为16或者32,如果有80个线程,那么肯定会带来太多不必要的线程上下文切换开销.

再来第二种简单的但不知是否可行的方法(N为CPU总核数):
1. 如果是CPU密集型应用,则线程池大小设置为N+1
2. 如果是IO密集型应用,则线程池大小设置为2N+1

**如果一台服务器上只部署这一个应用并且只有这一个线程池,那么这种估算或许合理,具体还需自行测试验证.**

接下来在这个文档：服务器性能IO优化 中发现一个估算公式：
> 最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目

比如平均每个线程CPU运行时间为0.5s,而线程等待时间（非CPU运行时间,比如IO）为1.5s,CPU核心数为8,那么根据上面这个公式估算得到：((0.5+1.5)/0.5)*8=32.这个公式进一步转化为：

> 最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目

可以得出一个结论：
**线程等待时间所占比例越高(IO密集型),需要越多线程.线程CPU时间所占比例越高(CPU密集型),需要越少线程.**

上一种估算方法也和这个结论相合.
一个系统最快的部分是CPU,所以决定一个系统吞吐量上限的是CPU。增强CPU处理能力,可以提高系统吞吐量上限。但根据短板效应,真实的系统吞吐量并不能单纯根据CPU来计算。那要提高系统吞吐量,就需要从“系统短板”（比如网络延迟、IO）着手：

1. 尽量提高短板操作的并行化比率,比如多线程下载技术
2. 增强短板能力,比如用NIO替代IO

第一条可以联系到Amdahl定律,这条定律定义了串行系统并行化后的加速比计算公式：

> 加速比=优化前系统耗时 / 优化后系统耗时

加速比越大,表明系统并行化的优化效果越好。Addahl定律还给出了系统并行度、CPU数目和加速比的关系,加速比为Speedup,系统串行化比率（指串行执行代码所占比率）为F,CPU数目为N

> Speedup <= 1 / (F + (1-F)/N)

当N足够大时,串行化比率F越小,加速比Speedup越大.

**是否使用线程池就一定比使用单线程高效呢?**

答案是否定的,比如Redis就是单线程的,但它却非常高效,基本操作都能达到十万量级/s。从线程这个角度来看,部分原因在于:
多线程带来线程上下文切换开销,单线程就没有这种开销锁

当然“Redis很快”更本质的原因在于：Redis基本都是内存操作,这种情况下单线程可以很高效地利用CPU。而多线程适用场景一般是：存在相当比例的IO和网络操作.

所以即使有上面的简单估算方法,也许看似合理,但实际上也未必合理,都需要结合系统真实情况（比如是IO密集型或者是CPU密集型或者是纯内存操作）和硬件环境（CPU、内存、硬盘读写速度、网络状况等）来不断尝试达到一个符合实际的合理估算值.

最后来一个“Dark Magic”估算方法(因为我暂时还没有搞懂它的原理),使用下面的类：

``` java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Timer;
import java.util.TimerTask;
import java.util.concurrent.BlockingQueue;

public abstract class PoolSizeCalculator {

    /**
     * The sample queue size to calculate the size of a single {@link Runnable} element.
     */
    private final int SAMPLE_QUEUE_SIZE = 1000;

    /**
     * Accuracy of test run. It must finish within 20ms of the testTime otherwise we retry the test. This could be
     * configurable.
     */
    private final int EPSYLON = 20;

    /**
     * Control variable for the CPU time investigation.
     */
    private volatile boolean expired;

    /**
     * Time (millis) of the test run in the CPU time calculation.
     */
    private final long testtime = 3000;

    /**
     * Calculates the boundaries of a thread pool for a given {@link Runnable}.
     *
     * @param targetUtilization    the desired utilization of the CPUs (0 <= targetUtilization <= 1)
     * @param targetQueueSizeBytes the desired maximum work queue size of the thread pool (bytes)
     */
    protected void calculateBoundaries(BigDecimal targetUtilization, BigDecimal targetQueueSizeBytes) {
        calculateOptimalCapacity(targetQueueSizeBytes);
        Runnable task = creatTask();
        start(task);
        start(task); // warm up phase
        long cputime = getCurrentThreadCPUTime();
        start(task); // test intervall
        cputime = getCurrentThreadCPUTime() - cputime;
        long waittime = (testtime * 1000000) - cputime;
        calculateOptimalThreadCount(cputime, waittime, targetUtilization);
    }

    private void calculateOptimalCapacity(BigDecimal targetQueueSizeBytes) {
        long mem = calculateMemoryUsage();
        BigDecimal queueCapacity = targetQueueSizeBytes.divide(new BigDecimal(mem), RoundingMode.HALF_UP);
        System.out.println("Target queue memory usage (bytes): " + targetQueueSizeBytes);
        System.out.println("createTask() produced " + creatTask().getClass().getName() + " which took " + mem
                + " bytes in a queue");
        System.out.println("Formula: " + targetQueueSizeBytes + " / " + mem);
        System.out.println("* Recommended queue capacity (bytes): " + queueCapacity);
    }

    /**
     * Brian Goetz' optimal thread count formula, see 'Java Concurrency in Practice' (chapter 8.2)
     *
     * @param cpu               cpu time consumed by considered task
     * @param wait              wait time of considered task
     * @param targetUtilization target utilization of the system
     */
    private void calculateOptimalThreadCount(long cpu, long wait, BigDecimal targetUtilization) {
        BigDecimal waitTime = new BigDecimal(wait);
        BigDecimal computeTime = new BigDecimal(cpu);
        BigDecimal numberOfCPU = new BigDecimal(Runtime.getRuntime().availableProcessors());
        BigDecimal optimalthreadcount = numberOfCPU.multiply(targetUtilization).multiply(
                new BigDecimal(1).add(waitTime.divide(computeTime, RoundingMode.HALF_UP)));
        System.out.println("Number of CPU: " + numberOfCPU);
        System.out.println("Target utilization: " + targetUtilization);
        System.out.println("Elapsed time (nanos): " + (testtime * 1000000));
        System.out.println("Compute time (nanos): " + cpu);
        System.out.println("Wait time (nanos): " + wait);
        System.out.println("Formula: " + numberOfCPU + " * " + targetUtilization + " * (1 + " + waitTime + " / "
                + computeTime + ")");
        System.out.println("* Optimal thread count: " + optimalthreadcount);
    }

    /**
     * Runs the {@link Runnable} over a period defined in {@link #testtime}. Based on Heinz Kabbutz' ideas
     * (http://www.javaspecialists.eu/archive/Issue124.html).
     *
     * @param task the runnable under investigation
     */
    public void start(Runnable task) {
        long start = 0;
        int runs = 0;
        do {
            if (++runs > 5) {
                throw new IllegalStateException("Test not accurate");
            }
            expired = false;
            start = System.currentTimeMillis();
            Timer timer = new Timer();
            timer.schedule(new TimerTask() {
                public void run() {
                    expired = true;
                }
            }, testtime);
            while (!expired) {
                task.run();
            }
            start = System.currentTimeMillis() - start;
            timer.cancel();
        } while (Math.abs(start - testtime) > EPSYLON);
        collectGarbage(3);
    }

    private void collectGarbage(int times) {
        for (int i = 0; i < times; i++) {
            System.gc();
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    /**
     * Calculates the memory usage of a single element in a work queue. Based on Heinz Kabbutz' ideas
     * (http://www.javaspecialists.eu/archive/Issue029.html).
     *
     * @return memory usage of a single {@link Runnable} element in the thread pools work queue
     */
    public long calculateMemoryUsage() {
        BlockingQueue<Runnable> queue = createWorkQueue();
        for (int i = 0; i < SAMPLE_QUEUE_SIZE; i++) {
            queue.add(creatTask());
        }
        long mem0 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        long mem1 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        queue = null;
        collectGarbage(15);
        mem0 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        queue = createWorkQueue();
        for (int i = 0; i < SAMPLE_QUEUE_SIZE; i++) {
            queue.add(creatTask());
        }
        collectGarbage(15);
        mem1 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        return (mem1 - mem0) / SAMPLE_QUEUE_SIZE;
    }

    /**
     * Create your runnable task here.
     *
     * @return an instance of your runnable task under investigation
     */
    protected abstract Runnable creatTask();

    /**
     * Return an instance of the queue used in the thread pool.
     *
     * @return queue instance
     */
    protected abstract BlockingQueue<Runnable> createWorkQueue();

    /**
     * Calculate current cpu time. Various frameworks may be used here, depending on the operating system in use. (e.g.
     * http://www.hyperic.com/products/sigar). The more accurate the CPU time measurement, the more accurate the results
     * for thread count boundaries.
     *
     * @return current cpu time of current thread
     */
    protected abstract long getCurrentThreadCPUTime();

}
```

然后自己继承这个抽象类并实现它的三个抽象方法,比如下面是我写的一个示例（任务是请求网络数据）,其中我指定期望CPU利用率为1.0（即100%）,任务队列总大小不超过100,000字节:
``` java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.lang.management.ManagementFactory;
import java.math.BigDecimal;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class SimplePoolSizeCaculatorImpl extends PoolSizeCalculator {

    @Override
    protected Runnable creatTask() {
        return new AsyncIOTask();
    }

    @Override
    protected BlockingQueue createWorkQueue() {
        return new LinkedBlockingQueue(1000);
    }

    @Override
    protected long getCurrentThreadCPUTime() {
        return ManagementFactory.getThreadMXBean().getCurrentThreadCpuTime();
    }

    public static void main(String[] args) {
        PoolSizeCalculator poolSizeCalculator = new SimplePoolSizeCaculatorImpl();
        poolSizeCalculator.calculateBoundaries(new BigDecimal(1.0), new BigDecimal(100000));
    }
}

/**
 * 自定义的异步IO任务
 * @author Will
 *
 */
class AsyncIOTask implements Runnable {

    @Override
    public void run() {
        HttpURLConnection connection = null;
        BufferedReader reader = null;
        try {
            String getURL = "http://www.baidu.com";
            URL getUrl = new URL(getURL);

            connection = (HttpURLConnection) getUrl.openConnection();
            connection.connect();
            reader = new BufferedReader(new InputStreamReader(
                    connection.getInputStream()));

            String line;
            while ((line = reader.readLine()) != null) {
                // empty loop
            }
        }

        catch (IOException e) {

        } finally {
            if(reader != null) {
                try {
                    reader.close();
                }
                catch(Exception e) {

                }
            }
            connection.disconnect();
        }

    }
}
```

得到的输出如下:
``` console
Target queue memory usage (bytes): 100000
createTask() produced com.example.demo.threadpool.AsyncIOTask which took 40 bytes in a queue
Formula: 100000 / 40
* Recommended queue capacity (bytes): 2500
Disconnected from the target VM, address: '127.0.0.1:57093', transport: 'socket'
Number of CPU: 8
Target utilization: 1
Elapsed time (nanos): 3000000000
Compute time (nanos): 93750000
Wait time (nanos): 2906250000
Formula: 8 * 1 * (1 + 2906250000 / 93750000)
* Optimal thread count: 256
```

推荐的任务队列大小为2500,线程数为256,有点出乎意料之外。我可以如下构造一个线程池：

``` java
ThreadPoolExecutor pool =
 new ThreadPoolExecutor(256, 256, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue(2500));
```

## 总结

Executors为我们提供了构造线程池的便捷方法,对于服务器程序我们应该杜绝使用这些便捷方法,而是直接使用线程池ThreadPoolExecutor的构造方法,避免无界队列可能导致的OOM以及线程个数限制不当导致的线程数耗尽等问题.ExecutorCompletionService提供了等待所有任务执行结束的有效方式,如果要设置等待的超时时间,则可以通过CountDownLatch完成.

通过对ThreadPoolExecutor源码的分析,从总体上了解了线程的创建,任务的添加,执行等过程,熟悉这些过程,使用线程池就会更新松了.
而从中学到的一些对并发控制,以及生产者--消费者模型任务处理器的使用,对以后理解或解决其他相关问题会有很大的帮助.比如Android中的Handler机制,而Looper中的Messager队列用一个BlookQueue来处理同样是可以的,这些就是读源码的收获吧