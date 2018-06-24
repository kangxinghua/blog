---
title: 显示锁（Lock）及Condition的学习与使用
date: 2018-06-24 10:54:11
tags:
---
synchronized是不错，但它并不完美。它有一些功能性的限制，比如:
* 它无法中断一个正在等候获得锁的线程，也无法通过投票得到锁。多线程竞争一个锁时，其余未得到锁的线程只能不停的尝试获得锁，而不能中断。高并发的情况下会导致性能下降。
* synchronized上是非公平的,新来的线程有可能立即获得监视器，而在等待区中等候已久的线程可能再次等待。

而Lock的一些实现类则很好的解决了这些问题。

<!-- more -->

## 可重入锁 ReentrantLock

java.util.concurrent.lock中的Lock框架是锁定的一个抽象，它允许把锁定的实现所为Java类，而不是作为语言的特性来实现。这就为Lock的多种实现留下了空间，各种实现可能不同的调度算法、性能特性或者锁定语义。

ReentrantLock类实现了Lock，它拥有与syschronized相同的并发性和内存语义，但是添加了类似锁投票、定时锁等候和可中断锁等候的一些特性。此外，它还提供了在激烈争用情况下更佳的性能。(换句句话，当许多线程都想访问共享资源时，JVM可以花更少的时候来调度线程，把更多时间用在执行线程上。)
``` java
class LockStudy {     
       
    private Lock lock = new ReentrantLock();// 锁对象   
             
    public void output(String name) {  
                        
        lock.lock();      // 得到锁       
        try {                   
            //doSomething            
        } finally {                  
            lock.unlock();// 释放锁                
        }            
    }        
}
```

需要注意的是，用syschronized修饰的方法或者语句块在代码执行完后锁自动释放，而用Lock需要我们手动释放锁，所以为了保证锁的最终被释放(发生异常情况)，需要互斥区放在try内，释放锁放在finally内。

## Condition

ReentrantLock里有个函数newCondition(),该函数得到一个锁上的“条件”，用于实现线程间的通信，条件变量很大一个程度上是为了解决Object.wait/notify/notifAll难以使用的问题。

Condition拥有await(),signalAll(),await对应Object.wait，signal对应于Object.notify，signalALL对应Object.notifyAll。特别说明的是Condition的接口改变名称就是为了避免与Object中的wait/notify/notifyAll的语义和使用上混淆，因为Condition同样有wait/notify/notifyAll方法()因为任何类都拥有这些方法。

每一个Lock可以有任意数据的Condition对象，Condition是与Lock绑定的，所以就有Lock的公平性的特性：如果是公平锁，线程为了按照FIFO(先进先出原则)的顺序从Condition.await中释放，如果是非公平锁，那么后续的锁竞争就不保证FIFO顺序了。下面是一个用Lock和Condition实现的一个生产者消费者的模式：
``` java
public class ProductQueue<T> {
        
    private final T[] items;
    private final Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();
    private int head, tail, count;
    
    public ProductQueue(int maxSize) {
        items = T[] new Object[maxSize];
    }
    
    public ProductQueue() {
        this(10);
    }
   
    public void put(T t) throws InterruptedException {
        lock.lock();
        try{
            while(count == getCapacity()) {
                notFull.await();
            }
            items[tail] = t;
            if(++tail==getCapacity()){
                tail = 0;
            }
            ++count;
            notEmpty.signalAll();
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while(count == 0) {
                notEmpty.await();
            }
            T ret = items[head];
            items[head] = null;//GC  
            if (++head == getCapacity()) {
                head = 0;
            }
            --count;
            notFull.signalAll();
            return ret;
        } finally {
            lock.unlock();
        }
    }
    public int getCapacity(){
        return items.length;
    }
    public int size() {
        lock.lock();
        try{
            return count;
        } finally{
            lock.unlock();
        }
    }
    
}
```

这个是多个Condition的强大之处，假设缓存队列中已经满了，那么阻塞的肯定是写线程，唤醒的肯定是读线程，相反，阻塞的肯定是读线程，唤醒的肯定是写线程，那么假设只有一个Condition会有什么效果，缓存队列已经存满，这个Lock不知道唤醒的是读线程还是写线程了，如果唤醒的是读线程，皆大欢喜，如果唤醒的是写线程，那么线程刚被唤醒又被阻塞了。这时又去唤醒，这样就很浪费时间。

## ReentrantLock与synchronized的对比

ReentrantLock同样是一个可重入锁，但与目前的 synchronized 实现相比，争用下的 ReentrantLock 实现更具可伸缩性。除了synchronized的功能,多了三个高级功能.

1. **等待可中断**
    在持有锁的线程长时间不释放锁的时候,等待的线程可以选择放弃等待.
    > tryLock(long timeout, TimeUnit unit)；
2. **公平锁**
    按照申请锁的顺序来一次获得锁称为公平锁.synchronized的是非公平锁,ReentrantLock可以通过构造函数实现公平锁.
    > new RenentrantLock(boolean fair)；
3. **绑定多个Condition**
    通过多次newCondition可以获得多个Condition对象,可以简单的实现比较复杂的线程同步的功能。通过await(),signal()等方法实现。

## Lock的其他实现类

如ReadWriteLock。ReentrantReadWriteLock实现了ReadWriteLock接口，构造器提供了公平锁和非公平锁两种创建方式。读-写锁定允许对共享数据进行更高级别的并发访问。虽然一次只有一个线程（writer 线程）可以修改共享数据，但在许多情况下，任何数量的线程可以同时读取共享数据（reader 线程）。读写锁适用于读多写少的情况，可以实现更好的并发性。