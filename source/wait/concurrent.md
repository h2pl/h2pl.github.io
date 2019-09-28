---
title: Java并发指南开篇：Java并发编程学习大纲
date: 2018-05-15 23:05:25
tags:
    - Java并发
categories:
	- 后端
	- Java并发
---

本文首发于我的个人博客：https://h2pl.github.io/

更多关于Java并发的文章请参阅我的CSDN专栏：Java并发指南 
https://blog.csdn.net/column/details/21961.html

相关代码会放在我的的Github：https://github.com/h2pl/

Java并发编程一直是Java程序员必须懂但又是很难懂的技术内容。

这里不仅仅是指使用简单的多线程编程，或者使用juc的某个类。当然这些都是并发编程的基本知识，除了使用这些工具以外，Java并发编程中涉及到的技术原理十分丰富。为了更好地把并发知识形成一个体系，也鉴于本人没有能力写出这类文章，于是参考几位并发编程专家的博客和书籍，做一个简单的整理和复习。

本文只是简要的介绍和总结。详细的内容欢迎来我的专栏阅读，会有更多的系列文章。

<!-- more -->

## 并发基础和多线程

首先需要学习的就是并发的基础知识，什么是并发，为什么要并发，多线程的概念，线程安全的概念等。

然后学会使用Java中的Thread或是其他线程实现方法，了解线程的状态转换，线程的方法，线程的通信方式等。

## JMM内存模型

任何语言最终都是运行在处理器上，JVM虚拟机为了给开发者一个一致的编程内存模型，需要制定一套规则，这套规则可以在不同架构的机器上有不同实现，并且向上为程序员提供统一的JMM内存模型。

所以了解JMM内存模型也是了解Java并发原理的一个重点，其中了解指令重排，内存屏障，以及可见性原理尤为重要。

JMM只保证happens-before和as-if-serial规则，所以在多线程并发时，可能出现原子性，可见性以及有序性这三大问题。

下面的内容则会讲述Java是如何解决这三大问题的。

## synchronized，volatile，final等关键字

对于并发的三大问题，volatile可以保证原子性和可见性，synchronized三种特性都可以保证（允许指令重排）。

synchronized是基于操作系统的mutex lock指令实现的，volatile和final则是根据JMM实现其内存语义。

此处还要了解CAS操作，它不仅提供了类似volatile的内存语义，并且保证操作原子性，因为它是由硬件实现的。

JUC中的Lock底层就是使用volatile加上CAS的方式实现的。synchronized也会尝试用cas操作来优化器重量级锁。

了解这些关键字是很有必要的。

## JUC包

在了解完上述内容以后，就可以看看JUC的内容了。

JUC提供了包括Lock，原子操作类，线程池，同步容器，工具类等内容。

这些类的基础都是AQS，所以了解AQS的原理是很重要的。

除此之外，还可以了解一下Fork/Join，以及JUC的常用场景，比如生产者消费者，阻塞队列，以及读写容器等。

## 实践

上述这些内容，除了JMM部分的内容比较不好实现之外，像是多线程基本使用，JUC的使用都可以在代码实践中更好地理解其原理。多尝试一些场景，或者在网上找一些比较经典的并发场景，或者参考别人的例子，在实践中加深理解，还是很有必要的。

## 补充

由于很多Java新手可能对并发编程没什么概念，在这里放一篇不错的总结，简要地提几个并发编程中比要重要的点，也是比较基本的点吗，算是抛砖引玉，开个好头，在大致了解了这些基础内容以后，才能更好地开展后面详细内容的学习。


![](https://upload-images.jianshu.io/upload_images/623504-3856f8a1e75a85c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)





### 1.并发编程三要素

*   原子性
    原子，即一个不可再被分割的颗粒。在Java中原子性指的是一个或多个操作要么全部执行成功要么全部执行失败。
*   有序性
    程序执行的顺序按照代码的先后顺序执行。（处理器可能会对指令进行重排序）
*   可见性
    当多个线程访问同一个变量时，如果其中一个线程对其作了修改，其他线程能立即获取到最新的值。

### 2\. 线程的五大状态

*   创建状态
    当用 new 操作符创建一个线程的时候
*   就绪状态
    调用 start 方法，处于就绪状态的线程并不一定马上就会执行 run 方法，还需要等待CPU的调度
*   运行状态
    CPU 开始调度线程，并开始执行 run 方法
*   阻塞状态
    线程的执行过程中由于一些原因进入阻塞状态
    比如：调用 sleep 方法、尝试去得到一个锁等等​​
*   死亡状态
    run 方法执行完 或者 执行过程中遇到了一个异常

### 3.悲观锁与乐观锁

*   悲观锁：每次操作都会加锁，会造成线程阻塞。
*   乐观锁：每次操作不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止，不会造成线程阻塞。​

### 4.线程之间的协作

#### 4.1 wait/notify/notifyAll

这一组是 Object 类的方法
需要注意的是：这三个方法都必须在同步的范围内调用​

*   wait
    阻塞当前线程，直到 notify 或者 notifyAll 来唤醒​​​​

    ```
    wait有三种方式的调用
    wait()
    必要要由 notify 或者 notifyAll 来唤醒​​​​
    wait(long timeout)
    在指定时间内，如果没有notify或notifAll方法的唤醒，也会自动唤醒。
    wait(long timeout,long nanos)
    本质上还是调用一个参数的方法
    public final void wait(long timeout, int nanos) throws InterruptedException {
          if (timeout < 0) {
                 throw new IllegalArgumentException("timeout value is negative");
           }
          if (nanos < 0 || nanos > 999999) {
                  throw new IllegalArgumentException(
                 "nanosecond timeout value out of range");
           }
           if (nanos > 0) {
                 timeout++;
           }
           wait(timeout);
    }
                  ​

    ```

    *   notify
        只能唤醒一个处于 wait 的线程
    *   notifyAll
        唤醒全部处于 wait 的线程
        ​

#### 4.2 sleep/yield/join

这一组是 Thread 类的方法

*   sleep
    让当前线程暂停指定时间，只是让出CPU的使用权，并不释放锁

*   yield
    暂停当前线程的执行，也就是当前CPU的使用权，让其他线程有机会执行，不能指定时间。会让当前线程从运行状态转变为就绪状态，此方法在生产环境中很少会使用到，​​​官方在其注释中也有相关的说明

    ```
          /**
          * A hint to the scheduler that the current thread is willing to yield
          * its current use of a processor. The scheduler is free to ignore this
          * hint.
          *
          * <p> Yield is a heuristic attempt to improve relative progression
          * between threads that would otherwise over-utilise a CPU. Its use
          * should be combined with detailed profiling and benchmarking to
          * ensure that it actually has the desired effect.
          *
          * <p> It is rarely appropriate to use this method. It may be useful
          * for debugging or testing purposes, where it may help to reproduce
          * bugs due to race conditions. It may also be useful when designing
          * concurrency control constructs such as the ones in the
          * {@link java.util.concurrent.locks} package.
          */​​
          ​​​​

    ```

*   join
    等待调用 join 方法的线程执行结束，才执行后面的代码
    其调用一定要在 start 方法之后（看源码可知）​
    使用场景：当父线程需要等待子线程执行结束才执行后面内容或者需要某个子线程的执行结果会用到 join 方法​

### 5.valitate 关键字

#### 5.1 定义

java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致的更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了volatile，在某些情况下比锁更加方便。如果一个字段被声明成volatile，java线程内存模型确保所有线程看到这个变量的值是一致的。

> valitate是轻量级的synchronized，不会引起线程上下文的切换和调度，执行开销更小。

#### 5.2 原理

1\. 使用volitate修饰的变量在汇编阶段，会多出一条lock前缀指令
2\. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成
3\. 它会强制将对缓存的修改操作立即写入主存
4\. 如果是写操作，它会导致其他CPU里缓存了该内存地址的数据无效

#### 5.3 作用

内存可见性
多线程操作的时候，一个线程修改了一个变量的值 ，其他线程能立即看到修改后的值
防止重排序
即程序的执行顺序按照代码的顺序执行（处理器为了提高代码的执行效率可能会对代码进行重排序）

> 并不能保证操作的原子性（比如下面这段代码的执行结果一定不是100000）

```
    public class testValitate {
    public volatile int inc = 0;
    public void increase() {
        inc = inc + 1;
    }
    public static void main(String[] args) {
        final testValitate test = new testValitate();
        for (int i = 0; i < 100; i++) {
            new Thread() {
                public void run() {
                    for (int j = 0; j < 1000; j++)
                        test.increase();
                }
            }.start();
        }
        while (Thread.activeCount() > 2) {  //保证前面的线程都执行完
            Thread.yield();
        }
        System.out.println(test.inc);
     }
   }

```

### 6\. synchronized 关键字

> 确保线程互斥的访问同步代码

#### 6.1 定义

synchronized 是JVM实现的一种锁，其中锁的获取和释放分别是
monitorenter 和 monitorexit 指令，该锁在实现上分为了偏向锁、轻量级锁和重量级锁，其中偏向锁在 java1.6 是默认开启的，轻量级锁在多线程竞争的情况下会膨胀成重量级锁，有关锁的数据都保存在对象头中

#### 6.2 原理

加了 synchronized 关键字的代码段，生成的字节码文件会多出 monitorenter 和 monitorexit 两条指令（利用javap -verbose 字节码文件可看到关，关于这两条指令的文档如下：

*   monitorenter
    Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
    • If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
    • If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
    • If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.​

*   monitorexit
    The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
    The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.​​

加了 synchronized 关键字的方法，生成的字节码文件中会多一个 ACC_SYNCHRONIZED 标志位，当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

#### 6.3 关于使用

*   修饰普通方法
    同步对象是实例对象
*   修饰静态方法
    同步对象是类本身
*   修饰代码块
    可以自己设置同步对象​

#### 6.4 缺点

会让没有得到锁的资源进入Block状态，争夺到资源之后又转为Running状态，这个过程涉及到操作系统用户模式和内核模式的切换，代价比较高。Java1.6为 synchronized 做了优化，增加了从偏向锁到轻量级锁再到重量级锁的过度，但是在最终转变为重量级锁之后，性能仍然较低。

### 7\. CAS

> AtomicBoolean，AtomicInteger，AtomicLong以及 Lock 相关类等底层就是用 CAS实现的，在一定程度上性能比 synchronized 更高。

#### 7.1 什么是CAS

CAS全称是Compare And Swap，即比较替换，是实现并发应用到的一种技术。操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。 如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值 。否则，处理器不做任何操作。

#### 7.2 为什么会有CAS

如果只是用 synchronized 来保证同步会存在以下问题
synchronized 是一种悲观锁，在使用上会造成一定的性能问题。在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。一个线程持有锁会导致其它所有需要此锁的线程挂起。

#### 7.3 实现原理

Java不能直接的访问操作系统底层，是通过native方法（JNI）来访问。CAS底层通过Unsafe类实现原子性操作。

#### 7.4 存在的问题

*   ABA问题
    什么是ABA问题？比如有一个 int 类型的值 N 是 1
    此时有三个线程想要去改变它：
    线程A ​​：希望给 N 赋值为 2
    线程B： 希望给 N 赋值为 2
    线程C： 希望给 N 赋值为 1​​
    此时线程A和线程B同时获取到N的值1，线程A率先得到系统资源，将 N 赋值为 2，线程 B 由于某种原因被阻塞住，线程C在线程A执行完后得到 N 的当前值2
    此时的线程状态
    线程A成功给 N 赋值为2
    线程B获取到 N 的当前值 1 希望给他赋值为 2，处于阻塞状态
    线程C获取当好 N 的当前值 2 ​​​​​希望给他赋值为1
    ​​
    然后线程C成功给N赋值为1
    ​最后线程B得到了系统资源，又重新恢复了运行状态，​在阻塞之前线程B获取到的N的值是1，执行compare操作发现当前N的值与获取到的值相同（均为1），成功将N赋值为了2。
    ​
    在这个过程中线程B获取到N的值是一个旧值​​，虽然和当前N的值相等，但是实际上N的值已经经历了一次 1到2到1的改变
    上面这个例子就是典型的ABA问题​
    怎样去解决ABA问题
    给变量加一个版本号即可，在比较的时候不仅要比较当前变量的值 还需要比较当前变量的版本号。Java中AtomicStampedReference 就解决了这个问题
*   循环时间长开销大
    在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很大的压力。

> CAS只能保证一个共享变量的原子操作

### 8\. AbstractQueuedSynchronizer(AQS)

AQS抽象的队列式同步器，是一种基于状态（state）的链表管理方式。state 是用CAS去修改的。它是 java.util.concurrent 包中最重要的基石，要学习想学习 java.util.concurrent 包里的内容这个类是关键。 ReentrantLock​、CountDownLatcher、Semaphore 实现的原理就是基于AQS。想知道他怎么实现以及实现原理 可以参看这篇文章[https://www.cnblogs.com/waterystone/p/4920797.html](https://link.jianshu.com/?t=https%3A%2F%2Fwww.cnblogs.com%2Fwaterystone%2Fp%2F4920797.html)

### 9\. Future

在并发编程我们一般使用Runable去执行异步任务，然而这样做我们是不能拿到异步任务的返回值的，但是使用Future 就可以。使用Future很简单，只需把Runable换成FutureTask即可。使用上比较简单，这里不多做介绍。

### 10\. 线程池

> 如果我们使用线程的时候就去创建一个线程，虽然简单，但是存在很大的问题。如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，这样频繁创建线程就会大大降低系统的效率，因为频繁创建线程和销毁线程需要时间。线程池通过复用可以大大减少线程频繁创建与销毁带来的性能上的损耗。

Java中线程池的实现类 ThreadPoolExecutor，其构造函数的每一个参数的含义在注释上已经写得很清楚了，这里几个关键参数可以再简单说一下

*   corePoolSize ：核心线程数即一直保留在线程池中的线程数量，即使处于闲置状态也不会被销毁。要设置 allowCoreThreadTimeOut 为 true，才会被销毁。
*   maximumPoolSize：线程池中允许存在的最大线程数
*   keepAliveTime ：非核心线程允许的最大闲置时间，超过这个时间就会本地销毁。
*   workQueue：用来存放任务的队列。
    *   SynchronousQueue：这个队列会让新添加的任务立即得到执行，如果线程池中所有的线程都在执行，那么就会去创建一个新的线程去执行这个任务。当使用这个队列的时候，maximumPoolSizes一般都会设置一个最大值 Integer.MAX_VALUE
    *   LinkedBlockingQueue：这个队列是一个无界队列。怎么理解呢，就是有多少任务来我们就会执行多少任务，如果线程池中的线程小于corePoolSize ,我们就会创建一个新的线程去执行这个任务，如果线程池中的线程数等于corePoolSize，就会将任务放入队列中等待，由于队列大小没有限制所以也被称为无界队列。当使用这个队列的时候 maximumPoolSizes 不生效（线程池中线程的数量不会超过corePoolSize），所以一般都会设置为0。
    *   ArrayBlockingQueue：这个队列是一个有界队列。可以设置队列的最大容量。当线程池中线程数大于或者等于 maximumPoolSizes 的时候，就会把任务放到这个队列中，当当前队列中的任务大于队列的最大容量就会丢弃掉该任务交由 RejectedExecutionHandler 处理。

最后，本文主要对Java并发编程开发需要的知识点作了简单的讲解，这里每一个知识点都可以用一篇文章去讲解，由于篇幅原因不能对每一个知识点都详细介绍，我相信通过本文你会对Java的并发编程会有更近一步的了解。如果您发现还有缺漏或者有错误的地方，可以在评论区补充，谢谢。