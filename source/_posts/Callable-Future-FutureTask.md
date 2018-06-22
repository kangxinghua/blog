---
title: Callable和Future、FutureTask的使用
date: 2018-06-21 18:19:09
tags: 笔记
---

在Java中，开启一个线程的唯一方式是，通过Thread的start方法，并且在线程中执行的Runnable的run方法。无论是线程池还是接下来要介绍的Callable，Future还是线程池，最核心最根本的还是调用的Thread.start()->Runnable.run(),其他的类的出现可以认为是更方便的使用Thread和Runnable，以此为核心更容易理解Java的并发框架。

<!-- more -->

虽然Thread和Runnable类使得多线程编程简单直接，但是有一个缺陷就是：在执行完任务之后无法获得执行结果。如果需要获得执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。因此从JDK1.5开始，有了一系列的类的出现来解决这些问题，如Callable和Future，FutureTask以及线程池《[从使用到原理学习Java线程池](/2018/05/04/ThreadPoolPrinciple/)》。

而自从Java 1.5开始，就是提供Callable和Future以及FutureTask，通过它们可以在任务执行完毕之后得到任务执行结果。

## 实现原理

**Thread和Runnable**

首先看Thread和Runnable的实现多线程任务的原理。
以下是简化后的代码，为了方便理解。
``` java
public class Thread implements Runnable {
    Runnable target;
	
	public Thread(Runnable runnable) {
	
		target = Runnable;
		 
		//省略其他初始化线程的任务
	}
	
	public void start() {
		 nativeCreate(this, stackSize, daemon);//native方法开启多线程，并调用run方法
	}
	
	public void run() {
	    if (target != null) {
	        target.run();
	    }
	}
}
```
可以看出target是一个Runnable对象，通过一个典型的装饰者模式来扩展Runnable，如果不传入，默认为null，需要自己实现run方法在新线程里任务，否则线程不会做任何事情就结束。所以无论如何怎么变化，最终都是Thread的start方法开启新的线程，run方法在这个新开启的线程执行任务，当然run方法也是可以单独调用，但是所在的线程是调用者的线程。

> 装饰者模式的典型特点：装饰后的类和被装饰的类，类型不变(继承Runnable)，提供新的行为，方法(start()等)。

**Callable和Future，FutureTask**

先通过URML图来看它们和Thread，Runnable之间的关系：
![Thread，Runnable之间的关系](006tKfTcjw1f762nmu5r7j30my0bydhp.jpg) 

*Callable*与Runnable的功能大致相似，Callable中有一个call()函数，**但是call()函数有返回值**，而Runnable的run()函数不能将结果返回给客户程序。

*Future*就是对Callable任务的执行结果进行取消、查询是否完成、获取结果、设置结果操作。其中的get()方法就是用来得到Callable的call()结果的。

*FutureTask*是Future的具体实现类，实现了get()等方法来对控制Callable的行为，又因为Thread只能执行Runnable，所以FutureTask实现了Runnable接口。

因为FutureTask需要在Thread中执行，所以需要在run()方法中完成具体的实现：
``` java
//简化后的代码，为了方便理解
public void run() {
    Callable<V> c = callable;
    
    if (c != null && state == NEW) {
        V result;
        result = c.call();
        set(result);
    }
}
```
通过get方法来获取结果，get()是哥阻塞方法，直到结果返回，或者中断发生。还可以通过get(long timeout, TimeUnit unit)方法控制等待结果的最大时间。

``` java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);//阻塞等待
    return report(s);
}
```

可以看出FutureTask的run方法实际的任务是在Callable的call中完成，FutureTask的实现方式采用了适配器模式来完成。

如果构造函数传入的是Runnable，则通过Executors的静态函数callable(Runnable task,...)将Runnable转换为Callable类型：

``` java
 public static  Callable callable(Runnable task, T result) { 
 
    if (task == null) throw new NullPointerException(); 
    
    return new RunnableAdapter(task, result); 
}
```

> 适配模式的典型特点：包装另一个对象(包装了Callable)，提供不同的接口(Runnable接口)

*Callable*和*Future*，*FutureTask*经常容易让人记忆混乱，理解后就知道了其实Future和FutureTask就是用来将Callable包装成一个Runnable,这样才能够在Thread中执行，同时提供将结果返回的功能，三个类总是同时出现，整体理解为是一个可以得到返回结果的Runnable。

## 使用

那么怎么使用这些类呢？由于FutureTask实现了Runnable，因此它既可以通过Thread包装来直接执行，也可以提交给ExecutorService来执行，在Thread中，就像使用Runnable一样。

示例代码如下：

``` java
public class FutureTest {    
    
    public static void main(String[] args) {                
        FutureTest futureTest = new FutureTest();            
        futureTest.useExecutor();            
        futureTest.useThread();  
    }  
      
    private void useExecutor() {      
      
        SumTask sumTask = new SumTask(1000);                
        ExecutorService executor = Executors.newCachedThreadPool();    
                
        FutureTask<Integer> futureTask = new FutureTask<Integer>(sumTask);    
                
        executor.submit(futureTask);        
        executor.shutdown();  
             
        try {            
            System.out.println(Thread.currentThread().getName()+"::useExecutor运行结果" + futureTask.get());        
        } catch (InterruptedException e) {            
            e.printStackTrace();      
        } catch (ExecutionException e) {            
            e.printStackTrace();       
        }    
    }  
     
    private void useThread() {        
        SumTask sumTask = new SumTask(500);          
            
        FutureTask<Integer> futureTask = new FutureTask<Integer>(sumTask) {    
            @Override    
            protected void done() {        
                super.done();        
                try {            
                // 这是在后台线程                   
                System.out.println(Thread.currentThread().getName()+"::useThread运行结果" + get());        
                } catch (InterruptedException e) {            
                    e.printStackTrace();        
                } catch (ExecutionException e) {            
                    e.printStackTrace();        
                }    
           }
        };          
          
        Thread thread = new Thread(futureTask);       
        thread.start();   
            
        try { 
        	  //这是在主线程，会阻塞           
            System.out.println(Thread.currentThread().getName()+"::useThread运行结果" + futureTask.get().getName());       
        } catch (InterruptedException e) {            
            e.printStackTrace();       
        } catch (ExecutionException e) {            
            e.printStackTrace();       
        }    
    }        
    
  class SumTask implements Callable<Integer> {        
      int number;   
           
      public SumTask(int num) {            
          this.number = num;      
      }       
       
      @Override       
      public Integer call() throws Exception {
                  
          System.out.println(Thread.currentThread().getName());            
          Thread.sleep(5000);  
                    
          int sum = 0;            
          for (int i = 0; i < number; i++) {                
              sum += i;            
          }           
          return sum;        
      }    
   }
}
```

结果：

> pool-1-thread-1
> main::useExecutor运行结果499500
> Thread-0
> main::useThread运行结果124750
> Thread-0::useThread运行结果124750

FutureTask.get()是阻塞的，useExecutor()和useThread()也会阻塞。这里只是说明FutureTask.get()所在的线程是调用者所在的线程，在Android中使用的话，一般是在FutureTask的done方法中get，这时get就是在后台线程调用了，然后通过Handler通知到UI或其他线程。