---
title: Java基础16：Java多线程基础最全总结
date: 2018-05-04 00:14:12
tags:
categories:
	- 后端
	- Java基础
---
本文介绍了Java多线程的基本概念，使用方法，以及底层实现原理。帮助你更好地使用Java的多线程。

具体代码在我的GitHub中可以找到
> https://github.com/h2pl/MyTech

喜欢的话麻烦点一下星哈谢谢。

文章首发于我的个人博客：
> https://h2pl.github.io/2018/05/04/javase16

更多关于Java后端学习的内容请到我的CSDN博客上查看：
> https://blog.csdn.net/a724888

<!-- more -->

## Java中的线程

Java之父对线程的定义是：
> 线程是一个独立执行的调用序列，同一个进程的线程在同一时刻共享一些系统资源（比如文件句柄等）也能访问同一个进程所创建的对象资源（内存资源）。java.lang.Thread对象负责统计和控制这种行为。

> 每个程序都至少拥有一个线程-即作为Java虚拟机(JVM)启动参数运行在主类main方法的线程。在Java虚拟机初始化过程中也可能启动其他的后台线程。这种线程的数目和种类因JVM的实现而异。然而所有用户级线程都是显式被构造并在主线程或者是其他用户线程中被启动。

      本文主要讲了java中多线程的使用方法、线程同步、线程数据传递、线程状态及相应的一些线程函数用法、概述等。在这之前，首先让我们来了解下在操作系统中进程和线程的区别：
    
    　　进程：每个进程都有独立的代码和数据空间（进程上下文），进程间的切换会有较大的开销，一个进程包含1--n个线程。（进程是资源分配的最小单位）
    
    　　线程：同一类线程共享代码和数据空间，每个线程有独立的运行栈和程序计数器(PC)，线程切换开销小。（线程是cpu调度的最小单位）
    
    　　线程和进程一样分为五个阶段：创建、就绪、运行、阻塞、终止。
    
    　　多进程是指操作系统能同时运行多个任务（程序）。
    
    　　多线程是指在同一程序中有多个顺序流在执行。
    
    在java中要想实现多线程，有两种手段，一种是继续Thread类，另外一种是实现Runable接口.(其实准确来讲，应该有三种，还有一种是实现Callable接口，并与Future、线程池结合使用

## Java线程内存模型
下面的图大致介绍了Java线程的调用过程，每个线程使用一个独立的调用栈进行线程执行，栈中的数据不共享，堆区和方法区的数据是共享的。
![image](http://incdn1.b0.upaiyun.com/2017/10/0daf3c6197b0a14eef74a013a154024a.png)

![image](http://incdn1.b0.upaiyun.com/2017/10/0daf3c6197b0a14eef74a013a154024a.png)

![image](http://incdn1.b0.upaiyun.com/2017/10/3d9d0af74829fa666dc137ef89a7b332.png)

## 构造方法和守护线程

    构造方法
    Thread类中不同的构造方法接受如下参数的不同组合：
    
    一个Runnable对象，这种情况下，Thread.start方法将会调用对应Runnable对象的run方法。如果没有提供Runnable对象，那么就会立即得到一个Thread.run的默认实现。
    
    一个作为线程标识名的String字符串，该标识在跟踪和调试过程中会非常有用，除此别无它用。
    
    线程组（ThreadGroup），用来放置新创建的线程，如果提供的ThreadGroup不允许被访问，那么就会抛出一个SecurityException 。
    
  
    
    Thread对象拥有一个守护(daemon)标识属性，这个属性无法在构造方法中被赋值，但是可以在线程启动之前设置该属性(通过setDaemon方法)。
    
    当程序中所有的非守护线程都已经终止，调用setDaemon方法可能会导致虚拟机粗暴的终止线程并退出。
    
    isDaemon方法能够返回该属性的值。守护状态的作用非常有限，即使是后台线程在程序退出的时候也经常需要做一些清理工作。
    
    （daemon的发音为”day-mon”,这是系统编程传统的遗留，系统守护进程是一个持续运行的进程，比如打印机队列管理，它总是在系统中运行。）
    
## 启动线程的方式和isAlive方法

启动线程
调用start方法会触发Thread实例以一个新的线程启动其run方法。新线程不会持有调用线程的任何同步锁。

当一个线程正常地运行结束或者抛出某种未检测的异常（比如，运行时异常(RuntimeException)，错误(ERROR) 或者其子类）线程就会终止。

**当线程终止之后，是不能被重新启动的。在同一个Thread上调用多次start方法会抛出InvalidThreadStateException异常。**

如果线程已经启动但是还没有终止，那么调用isAlive方法就会返回true.即使线程由于某些原因处于阻塞(Blocked)状态该方法依然返回true。

如果线程已经被取消(cancelled),那么调用其isAlive在什么时候返回false就因各Java虚拟机的实现而异了。没有方法可以得知一个处于非活动状态的线程是否已经被启动过了。

## 优先级

**Java的线程实现基本上都是内核级线程的实现，所以Java线程的具体执行还取决于操作系统的特性。**

Java虚拟机为了实现跨平台(不同的硬件平台和各种操作系统)的特性，Java语言在线程调度与调度公平性上未作出任何的承诺，甚至都不会严格保证线程会被执行。但是Java线程却支持优先级的方法，这些方法会影响线程的调度：

每个线程都有一个优先级，分布在Thread.MIN_PRIORITY和Thread.MAX_PRIORITY之间（分别为1和10）
默认情况下，新创建的线程都拥有和创建它的线程相同的优先级。main方法所关联的初始化线程拥有一个默认的优先级，这个优先级是Thread.NORM_PRIORITY (5).

线程的当前优先级可以通过getPriority方法获得。
线程的优先级可以通过setPriority方法来动态的修改，一个线程的最高优先级由其所在的线程组限定。

## 线程的控制方法

只有很少几个方法可以用于跨线程交流：

    每个线程都有一个相关的Boolean类型的中断标识。在线程t上调用t.interrupt会将该线程的中断标识设为true，除非线程t正处于Object.wait,Thread.sleep,或者Thread.join,这些情况下interrupt调用会导致t上的这些操作抛出InterruptedException异常，但是t的中断标识会被设为false。
    
    任何一个线程的中断状态都可以通过调用isInterrupted方法来得到。如果线程已经通过interrupt方法被中断，这个方法将会返回true。
    
    但是如果调用了Thread.interrupted方法且中断标识还没有被重置，或者是线程处于wait，sleep，join过程中，调用isInterrupted方法将会抛出InterruptedException异常。
    
    调用t.join()方法将会暂停执行调用线程，直到线程t执行完毕：当t.isAlive()方法返回false的时候调用t.join()将会直接返回(return)。
    
    另一个带参数毫秒(millisecond)的join方法在被调用时，如果线程没能够在指定的时间内完成，调用线程将重新得到控制权。
    
    因为isAlive方法的实现原理，所以在一个还没有启动的线程上调用join方法是没有任何意义的。同样的，试图在一个还没有创建的线程上调用join方法也是不明智的。
    
    起初，Thread类还支持一些另外一些控制方法：suspend,resume,stop以及destroy。这几个方法已经被声明过期。其中destroy方法从来没有被实现，估计以后也不会。而通过使用等待/唤醒机制增加suspend和resume方法在安全性和可靠性的效果有所欠缺
## Thread的静态方法

    静态方法
    Thread类中的部分方法被设计为只适用于当前正在运行的线程（即调用Thread方法的线程）。为强调这点，这些方法都被声明为静态的。
    
    Thread.currentThread方法会返回当前线程的引用，得到这个引用可以用来调用其他的非静态方法，比如Thread.currentThread().getPriority()会返回调用线程的优先级。
    
    Thread.interrupted方法会清除当前线程的中断状态并返回前一个状态。（一个线程的中断状态是不允许被其他线程清除的）
    
    Thread.sleep(long msecs)方法会使得当前线程暂停执行至少msecs毫秒。
    
    Thread.yield方法纯粹只是建议Java虚拟机对其他已经处于就绪状态的线程（如果有的话）调度执行，而不是当前线程。最终Java虚拟机如何去实现这种行为就完全看其喜好了。
    
## 线程组

    每一个线程都是一个线程组中的成员。默认情况下，新建线程和创建它的线程属于同一个线程组。线程组是以树状分布的。
    
    当创建一个新的线程组，这个线程组成为当前线程组的子组。getThreadGroup方法会返回当前线程所属的线程组，对应地，ThreadGroup类也有方法可以得到哪些线程目前属于这个线程组，比如enumerate方法。

    ThreadGroup类存在的一个目的是支持安全策略来动态的限制对该组的线程操作。比如对不属于同一组的线程调用interrupt是不合法的。
    
    这是为避免某些问题(比如，一个applet线程尝试杀掉主屏幕的刷新线程)所采取的措施。ThreadGroup也可以为该组所有线程设置一个最大的线程优先级。

    线程组往往不会直接在程序中被使用。在大多数的应用中，如果仅仅是为在程序中跟踪线程对象的分组，那么普通的集合类（比如java.util.Vector）应是更好的选择。

## 多线程的实现

    public class 多线程实例 {
    
        //继承thread
        @Test
        public void test1() {
            class A extends Thread {
                @Override
                public void run() {
                    System.out.println("A run");
                }
            }
            A a = new A();
            a.start();
        }
    
        //实现Runnable
        @Test
        public void test2() {
            class B implements Runnable {
    
                @Override
                public void run() {
                    System.out.println("B run");
                }
            }
            B b = new B();
            //Runable实现类需要由Thread类包装后才能执行
            new Thread(b).start();
        }
    
        //有返回值的线程
        @Test
        public void test3() {
            Callable callable = new Callable() {
                int sum = 0;
                @Override
                public Object call() throws Exception {
                    for (int i = 0;i < 5;i ++) {
                        sum += i;
                    }
                    return sum;
                }
            };
            //这里要用FutureTask，否则不能加入Thread构造方法
            FutureTask futureTask = new FutureTask(callable);
            new Thread(futureTask).start();
            try {
                System.out.println(futureTask.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    
        //线程池实现
        @Test
        public void test4() {
            ExecutorService executorService = Executors.newFixedThreadPool(5);
            //execute直接执行线程
            executorService.execute(new Thread());
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println("runnable");
                }
            });
            //submit提交有返回结果的任务，运行完后返回结果。
            Future future = executorService.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    return "a";
                }
            });
            try {
                System.out.println(future.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
    
            ArrayList<String> list = new ArrayList<>();
            //有返回值的线程组将返回值存进集合
            for (int i = 0;i < 5;i ++ ) {
                int finalI = i;
                Future future1 = executorService.submit(new Callable<String>() {
                    @Override
                    public String call() throws Exception {
                        return "res" + finalI;
                    }
                });
                try {
                    list.add((String) future1.get());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
            }
            for (String s : list) {
                System.out.println(s);
            }
        }
    }

## 线程状态转换

    public class 线程的状态转换 {
    //一开始线程是init状态，结束时是terminated状态
    class t implements Runnable {
        private String name;
        public t(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            System.out.println(name + "run");
        }
    }

    //测试join，父线程在子线程运行时进入waiting状态
    @Test
    public void test1() throws InterruptedException {
        Thread dad = new Thread(new Runnable() {
            Thread son = new Thread(new t("son"));
            @Override
            public void run() {
                System.out.println("dad init");
                son.start();
                try {
                    //保证子线程运行完再运行父线程
                    son.join();
                    System.out.println("dad run");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        //调用start，线程进入runnable状态，等待系统调度
        dad.start();
        //在父线程中对子线程实例使用join，保证子线程在父线程之前执行完

    }

    //测试sleep
    @Test
    public void test2(){
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("t1 run");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        //主线程休眠。进入time waiting状态
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t1.start();

    }

    //线程2进入blocked状态。
    public static void main(String[] args) {
        test4();
        Thread.yield();//进入runnable状态
    }

    //测试blocked状态
    public static void test4() {
        class A {
            //线程1获得实例锁以后线程2无法获得实例锁，所以进入blocked状态
            synchronized void run() {
                while (true) {
                    System.out.println("run");
                }
            }
        }
        A a = new A();
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("t1 get lock");
                a.run();
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("t2 get lock");
                a.run();
            }
        }).start();

    }

    //volatile保证线程可见性
    volatile static int flag = 1;
    //object作为锁对象，用于线程使用wait和notify方法
    volatile static Object o = new Object();
    //测试wait和notify
    //wait后进入waiting状态，被notify进入blocked（阻塞等待锁释放）或者runnable状态（获取到锁）
    public void test5() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                //wait和notify只能在同步代码块内使用
                synchronized (o) {
                    while (true) {
                        if (flag == 0) {
                            try {
                                Thread.sleep(2000);
                                System.out.println("thread1 wait");
                                //释放锁，线程挂起进入object的等待队列，后续代码运行
                                o.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                        System.out.println("thread1 run");
                        System.out.println("notify t2");
                        flag = 0;
                        //通知等待队列的一个线程获取锁
                        o.notify();
                    }
                }
            }
        }).start();
        //解释同上
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    synchronized (o) {
                        if (flag == 1) {
                            try {
                                Thread.sleep(2000);
                                System.out.println("thread2 wait");
                                o.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                        System.out.println("thread2 run");
                        System.out.println("notify t1");
                        flag = 1;
                        o.notify();
                    }
                }
            }
        }).start();
    }

    //输出结果是
    //    thread1 run
    //    notify t2
    //    thread1 wait
    //    thread2 run
    //    notify t1
    //    thread2 wait
    //    thread1 run
    //    notify t2
    //不断循环

    }


