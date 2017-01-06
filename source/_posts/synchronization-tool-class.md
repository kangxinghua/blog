---
title: 同步工具类
date: 2017-01-05 14:30:33
tags: Java 并发编程实战
---
## 闭锁
&emsp;&emsp;闭锁可以延迟线程的进度直到其达到终止状态[CPJ 3.4.2]。闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能通过，当到达结束状态时，这扇门会打开并允许所有线程通过。当闭锁到达结束状态后，将不会再改变状态。因此这扇门将永远保持打开状态。闭锁可以用来确保某些活动直到其他活动都完成后才继续执行，例如:
- 确保某个计算在其需要的所有资源都被初始化之后才继续执行。二元闭锁（包括两个状态）可以用来表示“资源R已经被初始化”，而所有需要R的操作都必须先在这个闭锁上等待。
- 确保某个服务在其依赖的所有其他服务都已经启动之后才启动。每个服务都有一个相关的二元闭锁。当启动服务S时，将首先在S依赖的其他服务的闭锁上等待，在所有依赖的服务都启动后会释放闭锁S，这样其依赖S的服务才能继续执行。
- 等待直到某个操作的所有参与者（例如，在多玩家游戏中的所有玩家）都就绪再继续执行。在这种情况中，当所有玩家都准备就绪时，闭锁将到达结束状态。  

&emsp;&emsp;CountDownLatch是一种灵活的闭锁实现，可以在上述各种情况中使用，它可以使一个或多个线程等待一组事件发生。闭锁状态包括一个计数器，该计数器被初始化为一个正数，表示需要等待的事件数量。countDown方法递减计数器，表示有一个事件已经发生了，而await方法等待计数器达到零，这表示所有需要等待的事件都已经发生。如果计数器的值非零，那么await会一直阻塞到计数器为零，或者等待中的线程中断，或者等待超时。 
### 代码例子：
``` Java
/**
 * Created by KangXinghua on 2017/1/5.
 */
public class TestHarness {
    public long timeTakes(int nThreads, final Runnable task) throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }
            };
            t.start();
        }

        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }

    public static void main(String[] args) throws InterruptedException {
        TestHarness testHarness = new TestHarness();
        long nanoTime = testHarness.timeTakes(10, new Runnable() {
            public void run() {
                try {
                    Thread.sleep(1000);
                    System.out.println("Task is over!!!");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        System.out.println(nanoTime);
    }
}
```

## FutureTask
&emsp;&emsp;FutureTask也可以做闭锁。FutureTask表示的计算是通过Callable来实现的，相当于一种可生成结果的Runnable，并且可以处于以下3种状态：等待运行（Waiting to run），正在运行（Running）和运行完成（Completed）。“执行完成”表示计算的所有可能结束方式，包括正常结束、由于取消而结束和由于异常而结束等。当FutureTask 进入完成状态后，它会永远停在这个状态上。   
&emsp;&emsp;FutureTask.get的行为取决于任务的状态。如果任务已经完成，那么get会立即返回结果，否则get将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。FutureTask将计算结果从执行计算的线程传递到获取这个结果的线程，而FutureTask的规范确保了这种传递过程能实现结果的安全发布。  
### 代码例子：
``` Java
/**
 * Created by KangXinghua on 2017/1/4.
 */
public class Preloader {
    private final FutureTask<String> futureTask = new FutureTask<String>(new Callable<String>() {
        public String call() throws Exception {
            Thread.sleep(10000);
            return "Task is over";
        }
    });
    private final Thread thread = new Thread(futureTask);

    public void start() {
        thread.start();
    }

    public String get() throws ExecutionException, InterruptedException {
        return futureTask.get();
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Preloader preloader = new Preloader();
        preloader.start();
        System.out.println(preloader.get());
    }
}
```
&emsp;&emsp;在Preloader中，当get方法抛出ExecutionException时，可能是以下三种情况之一：Callable抛出的受检查异常，RuntimeException，以及Error。我们必须对每种情况单独处理。

## 信号量
&emsp;&emsp;计数信号量（Counting Semaphore）用来控制同时访问的某个特定资源的操作数量，或者同时执行某个指定操作的数量[CPJ 3.4.1]。计算信号量还可以用来实现某种资源池，或者对容器施加边界。  
&emsp;&emsp;Semaphore中管理着一组虚拟许可（permit），许可的初始量可通过构造函数来指定。在执行操作时可以首先获得许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么acquire将阻塞直到有许可（或者直到被中断或者操作超时）。release方法将返回一个许可给信号量。计算信号量的一种简化形式是二值信号量，即初始化值为1的Semaphore。二值信号量可以用做互斥体（mutex），并具备不可重入的加锁语义：谁拥有这个唯一的许可，谁就拥有了互斥锁。  
### 代码例子：
``` Java
/**
 * Created by KangXinghua on 2017/1/5.
 */
public class TestSemaphore {
    public static void main(String[] args) {
        // 线程池
        ExecutorService exec = Executors.newCachedThreadPool();
        // 只能5个线程同时访问
        final Semaphore semp = new Semaphore(5);
        // 模拟20个客户端访问
        for (int index = 0; index < 20; index++) {
            final int NO = index;
            Runnable run = new Runnable() {
                public void run() {
                    try {
                        // 获取许可
                        semp.acquire();
                        System.out.println("Accessing: " + NO);
                        Thread.sleep((long) (Math.random() * 10000));
                        // 访问完后，释放
                        semp.release();
                        System.out.println("-----------------" + semp.availablePermits());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                }

            };
            exec.execute(run);
        }
        // 退出线程池
        exec.shutdown();
    }
}
```
## 栅栏
&emsp;&emsp;栅栏（Barrier）类似于闭锁，它能阻塞一组线程直到某个事件发生[CPJ 4.4.3]。闭锁是一次性对象，一旦进入最终状态，就不能被重置了。栅栏与闭锁的关键区别在于，所有线程必须同时达到栅栏位置，才能继续执行。闭锁用于等待事件，而栅栏用于等待其他线程。栅栏用于实现一些协议，例如几个家庭决定在某个地方集合：“所有人6：00在麦当劳碰头，到了以后要等其他人，之后再讨论下一步要做的事情。”   
&emsp;&emsp;CyclicBarrer可以使一定数量的参与方反复地在栅栏位置汇集，它在并行迭代算法中非常有用：这种算法通常将一个问题拆分一些列互相独立的子问题。如果所有线程都达到了栅栏位置，那么栅栏将打开，为此所有线程都被释放，而栅栏将被重置以便下次使用。如果对await的调用超时，或者await阻塞的线程被中断，那么栅栏就认为是打破了，所有阻塞的await调用都将终止并抛出BrokenBarrerException。如果成功通过栅栏，那么await将为每个线程返回一个唯一的到达索引号，我们可以利用这些索引来“选举”产生一个领导线程，并在下一次迭代中由该领导线程执行一些特殊的工作。CyclicBarrer还可以使你将一个栅栏操作传递给构造函数,这是Runnable，当成功通过栅栏时会（在一个子任务线程中）执行它，但在阻塞线程被释放之前是不能被执行的。  
&emsp;&emsp;在模拟程序中通常需要使用栅栏，例如某个步骤中的计算可以并行执行，但必须等到该步骤中的所有计算都执行完成才能进入下一个步骤。  
### 代码例子：
``` Java
/**
 * Created by KangXinghua on 2017/1/6.
 */
public class TestCyclicBarrier {
    public static void main(String[] args) {
        int N = 4;
        CyclicBarrier barrier = new CyclicBarrier(N);

//        CyclicBarrier barrier  = new CyclicBarrier(N,new Runnable() {
//            @Override
//            public void run() {
//                System.out.println("当前线程"+Thread.currentThread().getName());
//            }
//        });

        for (int i = 0; i < N; i++) {
            new Writer(barrier).start();
        }

        try {
            Thread.sleep(25000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("CyclicBarrier重用");

        for (int i = 0; i < N; i++) {
            new Writer(barrier).start();
        }
    }

    static class Writer extends Thread {
        private CyclicBarrier cyclicBarrier;

        public Writer(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            System.out.println("线程" + Thread.currentThread().getName() + "正在写入数据...");
            try {
                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                System.out.println("线程" + Thread.currentThread().getName() + "写入数据完毕，等待其他线程写入完毕");

                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "所有线程写入完毕，继续处理其他任务...");
        }
    }
}
```