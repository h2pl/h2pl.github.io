---
title: Java并发指南12：深度解读 java 线程池设计思想及源码实现
date: 2018-05-23 22:46:48
tags:
    - JUC
categories:
	- 后端
	- Java并发
---
本文首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Java并发指南 
https://blog.csdn.net/column/details/21961.html

相关代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->


深度解读 java 线程池设计思想及源码实现

转自

https://javadoop.com/2017/09/05/java-thread-pool/hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io

我相信大家都看过很多的关于线程池的文章，基本上也是面试必问的，好像我写这篇文章其实是没有什么意义的，不过，我相信你也和我一样，看了很多文章还是一知半解，甚至可能看了很多瞎说的文章。希望大家看过这篇文章以后，就可以完全掌握 Java 线程池了。

> 我发现好些人都是因为这篇文章来到本站的，希望这篇让人留下第一眼印象的文章能给你带来收获。

本文一大重点是源码解析，不过线程池设计思想以及作者实现过程中的一些巧妙用法是我想传达给读者的。本文还是会一行行关键代码进行分析，目的是为了让那些自己看源码不是很理解的同学可以得到参考。

线程池是非常重要的工具，如果你要成为一个好的工程师，还是得比较好地掌握这个知识。即使你为了谋生，也要知道，这基本上是面试必问的题目，而且面试官很容易从被面试者的回答中捕捉到被面试者的技术水平。

本文略长，建议在 pc 上阅读，边看文章边翻源码（Java7 和 Java8 都一样），建议想好好看的读者抽出至少 15 至 30 分钟的整块时间来阅读。当然，如果读者仅为面试准备，可以直接滑到最后的总结部分。

目录

*   [总览](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#%E6%80%BB%E8%A7%88)
*   [Executor 接口](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#Executor%20%E6%8E%A5%E5%8F%A3)
*   [ExecutorService](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#ExecutorService)
*   [FutureTask](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#FutureTask)
*   [AbstractExecutorService](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#AbstractExecutorService)
*   [ThreadPoolExecutor](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#ThreadPoolExecutor)
*   [Executors](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#Executors)
*   [总结](https://javadoop.com/2017/09/05/java-thread-pool/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io#%E6%80%BB%E7%BB%93)



## 总览

开篇来一些废话。下图是 java 线程池几个相关类的继承结构：

![1](https://javadoop.com/blogimages/java-thread-pool/1.jpg)

先简单说说这个继承结构，Executor 位于最顶层，也是最简单的，就一个 execute(Runnable runnable) 接口方法定义。

ExecutorService 也是接口，在 Executor 接口的基础上添加了很多的接口方法，所以一般来说我们会使用这个接口。

然后再下来一层是 AbstractExecutorService，从名字我们就知道，这是抽象类，这里实现了非常有用的一些方法供子类直接使用，之后我们再细说。

然后才到我们的重点部分 ThreadPoolExecutor 类，这个类提供了关于线程池所需的非常丰富的功能。

另外，我们还涉及到下图中的这些类：

![others](https://javadoop.com/blogimages/java-thread-pool/others.png)

同在并发包中的 Executors 类，类名中带字母 s，我们猜到这个是工具类，里面的方法都是静态方法，如以下我们最常用的用于生成 ThreadPoolExecutor 的实例的一些方法：

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

```

另外，由于线程池支持获取线程执行的结果，所以，引入了 Future 接口，RunnableFuture 继承自此接口，然后我们最需要关心的就是它的实现类 FutureTask。到这里，记住这个概念，在线程池的使用过程中，我们是往线程池提交任务（task），使用过线程池的都知道，我们提交的每个任务是实现了 Runnable 接口的，其实就是先将 Runnable 的任务包装成 FutureTask，然后再提交到线程池。这样，读者才能比较容易记住 FutureTask 这个类名：它首先是一个任务（Task），然后具有 Future 接口的语义，即可以在将来（Future）得到执行的结果。

当然，线程池中的 BlockingQueue 也是非常重要的概念，如果线程数达到 corePoolSize，我们的每个任务会提交到等待队列中，等待线程池中的线程来取任务并执行。这里的 BlockingQueue 通常我们使用其实现类 LinkedBlockingQueue、ArrayBlockingQueue 和 SynchronousQueue，每个实现类都有不同的特征，使用场景之后会慢慢分析。想要详细了解各个 BlockingQueue 的读者，可以参考我的前面的一篇对 BlockingQueue 的各个实现类进行详细分析的文章。

> 把事情说完整：除了上面说的这些类外，还有一个很重要的类，就是定时任务实现类 ScheduledThreadPoolExecutor，它继承自本文要重点讲解的 ThreadPoolExecutor，用于实现定时执行。不过本文不会介绍它的实现，我相信读者看完本文后可以比较容易地看懂它的源码。

以上就是本文要介绍的知识，废话不多说，开始进入正文。



## Executor 接口

```
/* 
 * @since 1.5
 * @author Doug Lea
 */
public interface Executor {
    void execute(Runnable command);
}

```

我们可以看到 Executor 接口非常简单，就一个 `void execute(Runnable command)` 方法，代表提交一个任务。为了让大家理解 java 线程池的整个设计方案，我会按照 Doug Lea 的设计思路来多说一些相关的东西。

我们经常这样启动一个线程：

```
new Thread(new Runnable(){
  // do something
}).start();

```

用了线程池 Executor 后就可以像下面这么使用：

```
Executor executor = anExecutor;
executor.execute(new RunnableTask1());
executor.execute(new RunnableTask2());

```

如果我们希望线程池同步执行每一个任务，我们可以这么实现这个接口：

```
class DirectExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();// 这里不是用的new Thread(r).start()，也就是说没有启动任何一个新的线程。
    }
}

```

我们希望每个任务提交进来后，直接启动一个新的线程来执行这个任务，我们可以这么实现：

```
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();  // 每个任务都用一个新的线程来执行
    }
}

```

我们再来看下怎么组合两个 Executor 来使用，下面这个实现是将所有的任务都加到一个 queue 中，然后从 queue 中取任务，交给真正的执行器执行，这里采用 synchronized 进行并发控制：

```
class SerialExecutor implements Executor {
    // 任务队列
    final Queue<Runnable> tasks = new ArrayDeque<Runnable>();
    // 这个才是真正的执行器
    final Executor executor;
    // 当前正在执行的任务
    Runnable active;

    // 初始化的时候，指定执行器
    SerialExecutor(Executor executor) {
        this.executor = executor;
    }

    // 添加任务到线程池: 将任务添加到任务队列，scheduleNext 触发执行器去任务队列取任务
    public synchronized void execute(final Runnable r) {
        tasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (active == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((active = tasks.poll()) != null) {
            // 具体的执行转给真正的执行器 executor
            executor.execute(active);
        }
    }
}

```

当然了，Executor 这个接口只有提交任务的功能，太简单了，我们想要更丰富的功能，比如我们想知道执行结果、我们想知道当前线程池有多少个线程活着、已经完成了多少任务等等，这些都是这个接口的不足的地方。接下来我们要介绍的是继承自 `Executor` 接口的 `ExecutorService` 接口，这个接口提供了比较丰富的功能，也是我们最常使用到的接口。



## ExecutorService

一般我们定义一个线程池的时候，往往都是使用这个接口：

```
ExecutorService executor = Executors.newFixedThreadPool(args...);
ExecutorService executor = Executors.newCachedThreadPool(args...);

```

因为这个接口中定义的一系列方法大部分情况下已经可以满足我们的需要了。

那么我们简单初略地来看一下这个接口中都有哪些方法：

```
public interface ExecutorService extends Executor {

    // 关闭线程池，已提交的任务继续执行，不接受继续提交新任务
    void shutdown();

    // 关闭线程池，尝试停止正在执行的所有任务，不接受继续提交新任务
    // 它和前面的方法相比，加了一个单词“now”，区别在于它会去停止当前正在进行的任务
    List<Runnable> shutdownNow();

    // 线程池是否已关闭
    boolean isShutdown();

    // 如果调用了 shutdown() 或 shutdownNow() 方法后，所有任务结束了，那么返回true
    // 这个方法必须在调用shutdown或shutdownNow方法之后调用才会返回true
    boolean isTerminated();

    // 等待所有任务完成，并设置超时时间
    // 我们这么理解，实际应用中是，先调用 shutdown 或 shutdownNow，
    // 然后再调这个方法等待所有的线程真正地完成，返回值意味着有没有超时
    boolean awaitTermination(long timeout, TimeUnit unit)
            throws InterruptedException;

    // 提交一个 Callable 任务
    <T> Future<T> submit(Callable<T> task);

    // 提交一个 Runnable 任务，第二个参数将会放到 Future 中，作为返回值，
    // 因为 Runnable 的 run 方法本身并不返回任何东西
    <T> Future<T> submit(Runnable task, T result);

    // 提交一个 Runnable 任务
    Future<?> submit(Runnable task);

    // 执行所有任务，返回 Future 类型的一个 list
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException;

    // 也是执行所有任务，但是这里设置了超时时间
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
            throws InterruptedException;

    // 只有其中的一个任务结束了，就可以返回，返回执行完的那个任务的结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
            throws InterruptedException, ExecutionException;

    // 同上一个方法，只有其中的一个任务结束了，就可以返回，返回执行完的那个任务的结果，
    // 不过这个带超时，超过指定的时间，抛出 TimeoutException 异常
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException;
}

```

这些方法都很好理解，一个简单的线程池主要就是这些功能，能提交任务，能获取结果，能关闭线程池，这也是为什么我们经常用这个接口的原因。



## FutureTask

在继续往下层介绍 ExecutorService 的实现类之前，我们先来说说相关的类 FutureTask。

```
Future   -> RunnableFuture -> FutureTask
Runnable -> RunnableFuture

FutureTask 通过 RunnableFuture 间接实现了 Runnable 接口，
所以每个 Runnable 通常都先包装成 FutureTask，
然后调用 executor.execute(Runnable command) 将其提交给线程池

```

我们知道，Runnable 的 void run() 方法是没有返回值的，所以，通常，如果我们需要的话，会在 submit 中指定第二个参数作为返回值：

```
<T> Future<T> submit(Runnable task, T result);

```

其实到时候会通过这两个参数，将其包装成 Callable。

Callable 也是因为线程池的需要，所以才有了这个接口。它和 Runnable 的区别在于 run() 没有返回值，而 Callable 的 call() 方法有返回值，同时，如果运行出现异常，call() 方法会抛出异常。

```
public interface Callable<V> {

    V call() throws Exception;
}

```

在这里，就不展开说 FutureTask 类了，因为本文篇幅本来就够大了，这里我们需要知道怎么用就行了。

下面，我们来看看 `ExecutorService` 的抽象实现 `AbstractExecutorService` 。



## AbstractExecutorService

AbstractExecutorService 抽象类派生自 ExecutorService 接口，然后在其基础上实现了几个实用的方法，这些方法提供给子类进行调用。

这个抽象类实现了 invokeAny 方法和 invokeAll 方法，这里的两个 newTaskFor 方法也比较有用，用于将任务包装成 FutureTask。定义于最上层接口 Executor中的 `void execute(Runnable command)` 由于不需要获取结果，不会进行 FutureTask 的包装。

> 需要获取结果（FutureTask），用 submit 方法，不需要获取结果，可以用 execute 方法。

下面，我将一行一行源码地来分析这个类，跟着源码来看看其实现吧：

> Tips: invokeAny 和 invokeAll 方法占了这整个类的绝大多数篇幅，读者可以选择适当跳过，因为它们可能在你的实践中使用的频次比较低，而且它们不带有承前启后的作用，不用担心会漏掉什么导致看不懂后面的代码。

```
public abstract class AbstractExecutorService implements ExecutorService {

    // RunnableFuture 是用于获取执行结果的，我们常用它的子类 FutureTask
    // 下面两个 newTaskFor 方法用于将我们的任务包装成 FutureTask 提交到线程池中执行
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    // 提交任务
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        // 1\. 将任务包装成 FutureTask
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 2\. 交给执行器执行，execute 方法由具体的子类来实现
        // 前面也说了，FutureTask 间接实现了Runnable 接口。
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        // 1\. 将任务包装成 FutureTask
        RunnableFuture<T> ftask = newTaskFor(task, result);
        // 2\. 交给执行器执行
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        // 1\. 将任务包装成 FutureTask
        RunnableFuture<T> ftask = newTaskFor(task);
        // 2\. 交给执行器执行
        execute(ftask);
        return ftask;
    }

    // 此方法目的：将 tasks 集合中的任务提交到线程池执行，任意一个线程执行完后就可以结束了
    // 第二个参数 timed 代表是否设置超时机制，超时时间为第三个参数，
    // 如果 timed 为 true，同时超时了还没有一个线程返回结果，那么抛出 TimeoutException 异常
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                            boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        // 任务数
        int ntasks = tasks.size();
        if (ntasks == 0)
            throw new IllegalArgumentException();
        // 
        List<Future<T>> futures= new ArrayList<Future<T>>(ntasks);

        // ExecutorCompletionService 不是一个真正的执行器，参数 this 才是真正的执行器
        // 它对执行器进行了包装，每个任务结束后，将结果保存到内部的一个 completionQueue 队列中
        // 这也是为什么这个类的名字里面有个 Completion 的原因吧。
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);
        try {
            // 用于保存异常信息，此方法如果没有得到任何有效的结果，那么我们可以抛出最后得到的一个异常
            ExecutionException ee = null;
            long lastTime = timed ? System.nanoTime() : 0;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // 首先先提交一个任务，后面的任务到下面的 for 循环一个个提交
            futures.add(ecs.submit(it.next()));
            // 提交了一个任务，所以任务数量减 1
            --ntasks;
            // 正在执行的任务数(提交的时候 +1，任务结束的时候 -1)
            int active = 1;

            for (;;) {
                // ecs 上面说了，其内部有一个 completionQueue 用于保存执行完成的结果
                // BlockingQueue 的 poll 方法不阻塞，返回 null 代表队列为空
                Future<T> f = ecs.poll();
                // 为 null，说明刚刚提交的第一个线程还没有执行完成
                // 在前面先提交一个任务，加上这里做一次检查，也是为了提高性能
                if (f == null) {
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    // 这里是 else if，不是 if。这里说明，没有任务了，同时 active 为 0 说明
                    // 任务都执行完成了。其实我也没理解为什么这里做一次 break？
                    // 因为我认为 active 为 0 的情况，必然从下面的 f.get() 返回了

                    // 2018-02-23 感谢读者 newmicro 的 comment，
                    //  这里的 active == 0，说明所有的任务都执行失败，那么这里是 for 循环出口
                    else if (active == 0)
                        break;
                    // 这里也是 else if。这里说的是，没有任务了，但是设置了超时时间，这里检测是否超时
                    else if (timed) {
                        // 带等待的 poll 方法
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        // 如果已经超时，抛出 TimeoutException 异常，这整个方法就结束了
                        if (f == null)
                            throw new TimeoutException();
                        long now = System.nanoTime();
                        nanos -= now - lastTime;
                        lastTime = now;
                    }
                    // 这里是 else。说明，没有任务需要提交，但是池中的任务没有完成，还没有超时(如果设置了超时)
                    // take() 方法会阻塞，直到有元素返回，说明有任务结束了
                    else
                        f = ecs.take();
                }
                /*
                 * 我感觉上面这一段并不是很好理解，这里简单说下。
                 * 1\. 首先，这在一个 for 循环中，我们设想每一个任务都没那么快结束，
                 *     那么，每一次都会进到第一个分支，进行提交任务，直到将所有的任务都提交了
                 * 2\. 任务都提交完成后，如果设置了超时，那么 for 循环其实进入了“一直检测是否超时”
                       这件事情上
                 * 3\. 如果没有设置超时机制，那么不必要检测超时，那就会阻塞在 ecs.take() 方法上，
                       等待获取第一个执行结果
                 * 4\. 如果所有的任务都执行失败，也就是说 future 都返回了，
                       但是 f.get() 抛出异常，那么从 active == 0 分支出去(感谢 newmicro 提出)
                         // 当然，这个需要看下面的 if 分支。
                 */

                // 有任务结束了
                if (f != null) {
                    --active;
                    try {
                        // 返回执行结果，如果有异常，都包装成 ExecutionException
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }// 注意看 for 循环的范围，一直到这里

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            // 方法退出之前，取消其他的任务
            for (Future<T> f : futures)
                f.cancel(true);
        }
    }

    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        try {
            return doInvokeAny(tasks, false, 0);
        } catch (TimeoutException cannotHappen) {
            assert false;
            return null;
        }
    }

    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return doInvokeAny(tasks, true, unit.toNanos(timeout));
    }

    // 执行所有的任务，返回任务结果。
    // 先不要看这个方法，我们先想想，其实我们自己提交任务到线程池，也是想要线程池执行所有的任务
    // 只不过，我们是每次 submit 一个任务，这里以一个集合作为参数提交
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            // 这个很简单
            for (Callable<T> t : tasks) {
                // 包装成 FutureTask
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                // 提交任务
                execute(f);
            }
            for (Future<T> f : futures) {
                if (!f.isDone()) {
                    try {
                        // 这是一个阻塞方法，直到获取到值，或抛出了异常
                        // 这里有个小细节，其实 get 方法签名上是会抛出 InterruptedException 的
                        // 可是这里没有进行处理，而是抛给外层去了。此异常发生于还没执行完的任务被取消了
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            // 这个方法返回，不像其他的场景，返回 List<Future>，其实执行结果还没出来
            // 这个方法返回是真正的返回，任务都结束了
            return futures;
        } finally {
            // 为什么要这个？就是上面说的有异常的情况
            if (!done)
                for (Future<T> f : futures)
                    f.cancel(true);
        }
    }

    // 带超时的 invokeAll，我们找不同吧
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null || unit == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));

            long lastTime = System.nanoTime();

            Iterator<Future<T>> it = futures.iterator();
            // 提交一个任务，检测一次是否超时
            while (it.hasNext()) {
                execute((Runnable)(it.next()));
                long now = System.nanoTime();
                nanos -= now - lastTime;
                lastTime = now;
                // 超时
                if (nanos <= 0)
                    return futures;
            }

            for (Future<T> f : futures) {
                if (!f.isDone()) {
                    if (nanos <= 0)
                        return futures;
                    try {
                        // 调用带超时的 get 方法，这里的参数 nanos 是剩余的时间，
                        // 因为上面其实已经用掉了一些时间了
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    long now = System.nanoTime();
                    nanos -= now - lastTime;
                    lastTime = now;
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (Future<T> f : futures)
                    f.cancel(true);
        }
    }

}

```

到这里，我们发现，这个抽象类包装了一些基本的方法，可是像 submit、invokeAny、invokeAll 等方法，它们都没有真正开启线程来执行任务，它们都只是在方法内部调用了 execute 方法，所以最重要的 execute(Runnable runnable) 方法还没出现，需要等具体执行器来实现这个最重要的部分，这里我们要说的就是 ThreadPoolExecutor 类了。

> 鉴于本文的篇幅，我觉得看到这里的读者应该已经不多了，快餐文化使然啊！我写的每篇文章都力求让读者可以通过我的一篇文章而记住所有的相关知识点，所以篇幅不免长了些。其实，工作了很多年的话，会有一个感觉，比如说线程池，即使看了 20 篇各种总结，也不如一篇长文实实在在讲解清楚每一个知识点，有点少即是多，多即是少的意味了。



## ThreadPoolExecutor

ThreadPoolExecutor 是 JDK 中的线程池实现，这个类实现了一个线程池需要的各个方法，它实现了任务提交、线程管理、监控等等方法。

我们可以基于它来进行业务上的扩展，以实现我们需要的其他功能，比如实现定时任务的类 ScheduledThreadPoolExecutor 就继承自 ThreadPoolExecutor。当然，这不是本文关注的重点，下面，还是赶紧进行源码分析吧。

首先，我们来看看线程池实现中的几个概念和处理流程。

我们先回顾下提交任务的几个方法：

```
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

```

一个最基本的概念是，submit 方法中，参数是 Runnable 类型（也有Callable 类型），这个参数不是用于 new Thread(runnable).start() 中的，此处的这个参数不是用于启动线程的，这里指的是任务，任务要做的事情是 run() 方法里面定义的或 Callable 中的 call() 方法里面定义的。

初学者往往会搞混这个，因为 Runnable 总是在各个地方出现，经常把一个 Runnable 包到另一个 Runnable 中。请把它想象成有个 Task 接口，这个接口里面有一个 run() 方法（我想作者只是不想因为这个再定义一个完全可以用 Runnable 来代替的接口，Callable 的出现，完全是因为 Runnable 不能满足需要）。

我们回过神来继续往下看，我画了一个简单的示意图来描述线程池中的一些主要的构件：

![pool-1](https://javadoop.com/blogimages/java-thread-pool/pool-1.png)

当然，上图没有考虑队列是否有界，提交任务时队列满了怎么办？什么情况下会创建新的线程？提交任务时线程池满了怎么办？空闲线程怎么关掉？这些问题下面我们会一一解决。

我们经常会使用 `Executors` 这个工具类来快速构造一个线程池，对于初学者而言，这种工具类是很有用的，开发者不需要关注太多的细节，只要知道自己需要一个线程池，仅仅提供必需的参数就可以了，其他参数都采用作者提供的默认值。

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

```

这里先不说有什么区别，它们最终都会导向这个构造方法：

```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        // 这几个参数都是必须要有的
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();

        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }

```

基本上，上面的构造方法中列出了我们最需要关心的几个属性了，下面逐个介绍下构造方法中出现的这几个属性：

*   corePoolSize

    > 核心线程数，不要抠字眼，反正先记着有这么个属性就可以了。

*   maximumPoolSize

    > ​最大线程数，线程池允许创建的最大线程数。

*   workQueue

    > 任务队列，BlockingQueue 接口的某个实现（常使用 ArrayBlockingQueue 和 LinkedBlockingQueue）。

*   keepAliveTime

    > 空闲线程的保活时间，如果某线程的空闲时间超过这个值都没有任务给它做，那么可以被关闭了。注意这个值并不会对所有线程起作用，如果线程池中的线程数少于等于核心线程数 corePoolSize，那么这些线程不会因为空闲太长时间而被关闭，当然，也可以通过调用 `allowCoreThreadTimeOut(true)`使核心线程数内的线程也可以被回收。

*   threadFactory

    > 用于生成线程，一般我们可以用默认的就可以了。通常，我们可以通过它将我们的线程的名字设置得比较可读一些，如 Message-Thread-1， Message-Thread-2 类似这样。

*   handler：

    > 当线程池已经满了，但是又有新的任务提交的时候，该采取什么策略由这个来指定。有几种方式可供选择，像抛出异常、直接拒绝然后返回等，也可以自己实现相应的接口实现自己的逻辑，这个之后再说。

除了上面几个属性外，我们再看看其他重要的属性。

Doug Lea 采用一个 32 位的整数来存放线程池的状态和当前池中的线程数，其中高 3 位用于存放线程池状态，低 29 位表示线程数（即使只有 29 位，也已经不小了，大概 5 亿多，现在还没有哪个机器能起这么多线程的吧）。我们知道，java 语言在整数编码上是统一的，都是采用补码的形式，下面是简单的移位操作和布尔操作，都是挺简单的。

```

private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

// 这里 COUNT_BITS 设置为 29(32-3)，意味着前三位用于存放线程状态，后29位用于存放线程数
// 很多初学者很喜欢在自己的代码中写很多 29 这种数字，或者某个特殊的字符串，然后分布在各个地方，这是非常糟糕的
private static final int COUNT_BITS = Integer.SIZE - 3;

// 000 11111111111111111111111111111
// 这里得到的是 29 个 1，也就是说线程池的最大线程数是 2^29-1=536870911
// 以我们现在计算机的实际情况，这个数量还是够用的
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 我们说了，线程池的状态存放在高 3 位中
// 运算结果为 111跟29个0：111 00000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
// 000 00000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 001 00000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
// 010 00000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
// 011 00000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// 将整数 c 的低 29 位修改为 0，就得到了线程池的状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 将整数 c 的高 3 为修改为 0，就得到了线程池中的线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }

private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */

private static boolean runStateLessThan(int c, int s) {
    return c < s;
}

private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}

```

上面就是对一个整数的简单的位操作，几个操作方法将会在后面的源码中一直出现，所以读者最好把方法名字和其代表的功能记住，看源码的时候也就不需要来来回回翻了。

在这里，介绍下线程池中的各个状态和状态变化的转换过程：

*   RUNNING：这个没什么好说的，这是最正常的状态：接受新的任务，处理等待队列中的任务
*   SHUTDOWN：不接受新的任务提交，但是会继续处理等待队列中的任务
*   STOP：不接受新的任务提交，不再处理等待队列中的任务，中断正在执行任务的线程
*   TIDYING：所有的任务都销毁了，workCount 为 0。线程池的状态在转换为 TIDYING 状态时，会执行钩子方法 terminated()
*   TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个

> RUNNING 定义为 -1，SHUTDOWN 定义为 0，其他的都比 0 大，所以等于 0 的时候不能提交任务，大于 0 的话，连正在执行的任务也需要中断。

看了这几种状态的介绍，读者大体也可以猜到十之八九的状态转换了，各个状态的转换过程有以下几种：

*   RUNNING -> SHUTDOWN：当调用了 shutdown() 后，会发生这个状态转换，这也是最重要的
*   (RUNNING or SHUTDOWN) -> STOP：当调用 shutdownNow() 后，会发生这个状态转换，这下要清楚 shutDown() 和 shutDownNow() 的区别了
*   SHUTDOWN -> TIDYING：当任务队列和线程池都清空后，会由 SHUTDOWN 转换为 TIDYING
*   STOP -> TIDYING：当任务队列清空后，发生这个转换
*   TIDYING -> TERMINATED：这个前面说了，当 terminated() 方法结束后

上面的几个记住核心的就可以了，尤其第一个和第二个。

另外，我们还要看看一个内部类 Worker，因为 Doug Lea 把线程池中的线程包装成了一个个 Worker，翻译成工人，就是线程池中做任务的线程。所以到这里，我们知道任务是 Runnable（内部叫 task 或 command），线程是 Worker。

Worker 这里又用到了抽象类 AbstractQueuedSynchronizer。题外话，AQS 在并发中真的是到处出现，而且非常容易使用，写少量的代码就能实现自己需要的同步方式（对 AQS 源码感兴趣的读者请参看我之前写的几篇文章）。

```
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    private static final long serialVersionUID = 6138294804551838833L;

    // 这个是真正的线程，任务靠你啦
    final Thread thread;

    // 前面说了，这里的 Runnable 是任务。为什么叫 firstTask？因为在创建线程的时候，如果同时指定了
    // 这个线程起来以后需要执行的第一个任务，那么第一个任务就是存放在这里的(线程可不止执行这一个任务)
    // 当然了，也可以为 null，这样线程起来了，自己到任务队列（BlockingQueue）中取任务（getTask 方法）就行了
    Runnable firstTask;

    // 用于存放此线程完全的任务数，注意了，这里用了 volatile，保证可见性
    volatile long completedTasks;

    // Worker 只有这一个构造方法，传入 firstTask，也可以传 null
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 调用 ThreadFactory 来创建一个新的线程
        this.thread = getThreadFactory().newThread(this);
    }

    // 这里调用了外部类的 runWorker 方法
    public void run() {
        runWorker(this);
    }

    ...// 其他几个方法没什么好看的，就是用 AQS 操作，来获取这个线程的执行权，用了独占锁
}

```

前面虽然啰嗦，但是简单。有了上面的这些基础后，我们终于可以看看 ThreadPoolExecutor 的 execute 方法了，前面源码分析的时候也说了，各种方法都最终依赖于 execute 方法：

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    // 前面说的那个表示 “线程池状态” 和 “线程数” 的整数
    int c = ctl.get();

    // 如果当前线程数少于核心线程数，那么直接添加一个 worker 来执行任务，
    // 创建一个新的线程，并把当前任务 command 作为这个线程的第一个任务(firstTask)
    if (workerCountOf(c) < corePoolSize) {
        // 添加任务成功，那么就结束了。提交任务嘛，线程池已经接受了这个任务，这个方法也就可以返回了
        // 至于执行的结果，到时候会包装到 FutureTask 中。
        // 返回 false 代表线程池不允许提交任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 到这里说明，要么当前线程数大于等于核心线程数，要么刚刚 addWorker 失败了

    // 如果线程池处于 RUNNING 状态，把这个任务添加到任务队列 workQueue 中
    if (isRunning(c) && workQueue.offer(command)) {
        /* 这里面说的是，如果任务进入了 workQueue，我们是否需要开启新的线程
         * 因为线程数在 [0, corePoolSize) 是无条件开启新的线程
         * 如果线程数已经大于等于 corePoolSize，那么将任务添加到队列中，然后进到这里
         */
        int recheck = ctl.get();
        // 如果线程池已不处于 RUNNING 状态，那么移除已经入队的这个任务，并且执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果线程池还是 RUNNING 的，并且线程数为 0，那么开启新的线程
        // 到这里，我们知道了，这块代码的真正意图是：担心任务提交到队列中了，但是线程都关闭了
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果 workQueue 队列满了，那么进入到这个分支
    // 以 maximumPoolSize 为界创建新的 worker，
    // 如果失败，说明当前线程数已经达到 maximumPoolSize，执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}

```

> 对创建线程的错误理解：如果线程数少于 corePoolSize，创建一个线程，如果线程数在 [corePoolSize, maximumPoolSize] 之间那么可以创建线程或复用空闲线程，keepAliveTime 对这个区间的线程有效。
> 
> 从上面的几个分支，我们就可以看出，上面的这段话是错误的。

上面这些一时半会也不可能全部消化搞定，我们先继续往下吧，到时候再回头看几遍。

这个方法非常重要 addWorker(Runnable firstTask, boolean core) 方法，我们看看它是怎么创建新的线程的：

```
// 第一个参数是准备提交给这个线程执行的任务，之前说了，可以为 null
// 第二个参数为 true 代表使用核心线程数 corePoolSize 作为创建线程的界线，也就说创建这个线程的时候，
//         如果线程池中的线程总数已经达到 corePoolSize，那么不能响应这次创建线程的请求
//         如果是 false，代表使用最大线程数 maximumPoolSize 作为界线
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 这个非常不好理解
        // 如果线程池已关闭，并满足以下条件之一，那么不创建新的 worker：
        // 1\. 线程池状态大于 SHUTDOWN，其实也就是 STOP, TIDYING, 或 TERMINATED
        // 2\. firstTask != null
        // 3\. workQueue.isEmpty()
        // 简单分析下：
        // 还是状态控制的问题，当线程池处于 SHUTDOWN 的时候，不允许提交任务，但是已有的任务继续执行
        // 当状态大于 SHUTDOWN 时，不允许提交任务，且中断正在执行的任务
        // 多说一句：如果线程池处于 SHUTDOWN，但是 firstTask 为 null，且 workQueue 非空，那么是允许创建 worker 的
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 如果成功，那么就是所有创建线程前的条件校验都满足了，准备创建线程执行任务了
            // 这里失败的话，说明有其他线程也在尝试往线程池中创建线程
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 由于有并发，重新再读取一下 ctl
            c = ctl.get();
            // 正常如果是 CAS 失败的话，进到下一个里层的for循环就可以了
            // 可是如果是因为其他线程的操作，导致线程池的状态发生了变更，如有其他线程关闭了这个线程池
            // 那么需要回到外层的for循环
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    /* 
     * 到这里，我们认为在当前这个时刻，可以开始创建线程来执行任务了，
     * 因为该校验的都校验了，至于以后会发生什么，那是以后的事，至少当前是满足条件的
     */

    // worker 是否已经启动
    boolean workerStarted = false;
    // 是否已将这个 worker 添加到 workers 这个 HashSet 中
    boolean workerAdded = false;
    Worker w = null;
    try {
        final ReentrantLock mainLock = this.mainLock;
        // 把 firstTask 传给 worker 的构造方法
        w = new Worker(firstTask);
        // 取 worker 中的线程对象，之前说了，Worker的构造方法会调用 ThreadFactory 来创建一个新的线程
        final Thread t = w.thread;
        if (t != null) {
            // 这个是整个类的全局锁，持有这个锁才能让下面的操作“顺理成章”，
            // 因为关闭一个线程池需要这个锁，至少我持有锁的期间，线程池不会被关闭
            mainLock.lock();
            try {

                int c = ctl.get();
                int rs = runStateOf(c);

                // 小于 SHUTTDOWN 那就是 RUNNING，这个自不必说，是最正常的情况
                // 如果等于 SHUTDOWN，前面说了，不接受新的任务，但是会继续执行等待队列中的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // worker 里面的 thread 可不能是已经启动的
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    // 加到 workers 这个 HashSet 中
                    workers.add(w);
                    int s = workers.size();
                    // largestPoolSize 用于记录 workers 中的个数的最大值
                    // 因为 workers 是不断增加减少的，通过这个值可以知道线程池的大小曾经达到的最大值
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 添加成功的话，启动这个线程
            if (workerAdded) {
                // 启动线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 如果线程没有启动，需要做一些清理工作，如前面 workCount 加了 1，将其减掉
        if (! workerStarted)
            addWorkerFailed(w);
    }
    // 返回线程是否启动成功
    return workerStarted;
}

```

简单看下 addWorkFailed 的处理：

```
// workers 中删除掉相应的 worker
// workCount 减 1
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        // rechecks for termination, in case the existence of this worker was holding up termination
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}

```

回过头来，继续往下走。我们知道，worker 中的线程 start 后，其 run 方法会调用 runWorker 方法：

```
// Worker 类的 run() 方法
public void run() {
    runWorker(this);
}

```

继续往下看 runWorker 方法：

```
// 此方法由 worker 线程启动后调用，这里用一个 while 循环来不断地从等待队列中获取任务并执行
// 前面说了，worker 在初始化的时候，可以指定 firstTask，那么第一个任务也就可以不需要从队列中获取
final void runWorker(Worker w) {
    // 
    Thread wt = Thread.currentThread();
    // 该线程的第一个任务(如果有的话)
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 循环调用 getTask 获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();          
            // 如果线程池状态大于等于 STOP，那么意味着该线程也要中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 这是一个钩子方法，留给需要的子类实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 到这里终于可以执行任务了
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    // 这里不允许抛出 Throwable，所以转换为 Error
                    thrown = x; throw new Error(x);
                } finally {
                    // 也是一个钩子方法，将 task 和异常作为参数，留给需要的子类实现
                    afterExecute(task, thrown);
                }
            } finally {
                // 置空 task，准备 getTask 获取下一个任务
                task = null;
                // 累加完成的任务数
                w.completedTasks++;
                // 释放掉 worker 的独占锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 如果到这里，需要执行线程关闭：
        // 1\. 说明 getTask 返回 null，也就是说，这个 worker 的使命结束了，执行关闭
        // 2\. 任务执行过程中发生了异常
        // 第一种情况，已经在代码处理了将 workCount 减 1，这个在 getTask 方法分析中会说
        // 第二种情况，workCount 没有进行处理，所以需要在 processWorkerExit 中处理
        // 限于篇幅，我不准备分析这个方法了，感兴趣的读者请自行分析源码
        processWorkerExit(w, completedAbruptly);
    }
}

```

我们看看 getTask() 是怎么获取任务的，这个方法写得真的很好，每一行都很简单，组合起来却所有的情况都想好了：

```
// 此方法有三种可能：
// 1\. 阻塞直到获取到任务返回。我们知道，默认 corePoolSize 之内的线程是不会被回收的，
//      它们会一直等待任务
// 2\. 超时退出。keepAliveTime 起作用的时候，也就是如果这么多时间内都没有任务，那么应该执行关闭
// 3\. 如果发生了以下条件，此方法必须返回 null:
//    - 池中有大于 maximumPoolSize 个 workers 存在(通过调用 setMaximumPoolSize 进行设置)
//    - 线程池处于 SHUTDOWN，而且 workQueue 是空的，前面说了，这种不再接受新的任务
//    - 线程池处于 STOP，不仅不接受新的线程，连 workQueue 中的线程也不再执行
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // 两种可能
        // 1\. rs == SHUTDOWN && workQueue.isEmpty()
        // 2\. rs >= STOP
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // CAS 操作，减少工作线程数
            decrementWorkerCount();
            return null;
        }

        boolean timed;      // Are workers subject to culling?
        for (;;) {
            int wc = workerCountOf(c);
            // 允许核心线程数内的线程回收，或当前线程数超过了核心线程数，那么有可能发生超时关闭
            timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // 这里 break，是为了不往下执行后一个 if (compareAndDecrementWorkerCount(c))
            // 两个 if 一起看：如果当前线程数 wc > maximumPoolSize，或者超时，都返回 null
            // 那这里的问题来了，wc > maximumPoolSize 的情况，为什么要返回 null？
            //    换句话说，返回 null 意味着关闭线程。
            // 那是因为有可能开发者调用了 setMaximumPoolSize 将线程池的 maximumPoolSize 调小了
            if (wc <= maximumPoolSize && ! (timedOut && timed))
                break;
            if (compareAndDecrementWorkerCount(c))
                return null;
            c = ctl.get();  // Re-read ctl
            // compareAndDecrementWorkerCount(c) 失败，线程池中的线程数发生了改变
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
        // wc <= maximumPoolSize 同时没有超时
        try {
            // 到 workQueue 中获取任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            // 如果此 worker 发生了中断，采取的方案是重试
            // 解释下为什么会发生中断，这个读者要去看 setMaximumPoolSize 方法，
            // 如果开发者将 maximumPoolSize 调小了，导致其小于当前的 workers 数量，
            // 那么意味着超出的部分线程要被关闭。重新进入 for 循环，自然会有部分线程会返回 null
            timedOut = false;
        }
    }
}

```

到这里，基本上也说完了整个流程，读者这个时候应该回到 execute(Runnable command) 方法，看看各个分支，我把代码贴过来一下：

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    // 前面说的那个表示 “线程池状态” 和 “线程数” 的整数
    int c = ctl.get();

    // 如果当前线程数少于核心线程数，那么直接添加一个 worker 来执行任务，
    // 创建一个新的线程，并把当前任务 command 作为这个线程的第一个任务(firstTask)
    if (workerCountOf(c) < corePoolSize) {
        // 添加任务成功，那么就结束了。提交任务嘛，线程池已经接受了这个任务，这个方法也就可以返回了
        // 至于执行的结果，到时候会包装到 FutureTask 中。
        // 返回 false 代表线程池不允许提交任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 到这里说明，要么当前线程数大于等于核心线程数，要么刚刚 addWorker 失败了

    // 如果线程池处于 RUNNING 状态，把这个任务添加到任务队列 workQueue 中
    if (isRunning(c) && workQueue.offer(command)) {
        /* 这里面说的是，如果任务进入了 workQueue，我们是否需要开启新的线程
         * 因为线程数在 [0, corePoolSize) 是无条件开启新的线程
         * 如果线程数已经大于等于 corePoolSize，那么将任务添加到队列中，然后进到这里
         */
        int recheck = ctl.get();
        // 如果线程池已不处于 RUNNING 状态，那么移除已经入队的这个任务，并且执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果线程池还是 RUNNING 的，并且线程数为 0，那么开启新的线程
        // 到这里，我们知道了，这块代码的真正意图是：担心任务提交到队列中了，但是线程都关闭了
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果 workQueue 队列满了，那么进入到这个分支
    // 以 maximumPoolSize 为界创建新的 worker，
    // 如果失败，说明当前线程数已经达到 maximumPoolSize，执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}

```

上面各个分支中，有两种情况会调用 reject(command) 来处理任务，因为按照正常的流程，线程池此时不能接受这个任务，所以需要执行我们的拒绝策略。接下来，我们说一说 ThreadPoolExecutor 中的拒绝策略。

```
final void reject(Runnable command) {
    // 执行拒绝策略
    handler.rejectedExecution(command, this);
}

```

此处的 handler 我们需要在构造线程池的时候就传入这个参数，它是 RejectedExecutionHandler 的实例。

RejectedExecutionHandler 在 ThreadPoolExecutor 中有四个已经定义好的实现类可供我们直接使用，当然，我们也可以实现自己的策略，不过一般也没有必要。

```
// 只要线程池没有被关闭，那么由提交任务的线程自己来执行这个任务。
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}

// 不管怎样，直接抛出 RejectedExecutionException 异常
// 这个是默认的策略，如果我们构造线程池的时候不传相应的 handler 的话，那就会指定使用这个
public static class AbortPolicy implements RejectedExecutionHandler {
    public AbortPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}

// 不做任何处理，直接忽略掉这个任务
public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}

// 这个相对霸道一点，如果线程池没有被关闭的话，
// 把队列队头的任务(也就是等待了最长时间的)直接扔掉，然后提交这个任务到等待队列中
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public DiscardOldestPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}

```

到这里，ThreadPoolExecutor 的源码算是分析结束了。单纯从源码的难易程度来说，ThreadPoolExecutor 的源码还算是比较简单的，只是需要我们静下心来好好看看罢了。

## 结束线程池的相关方法

tryTerminate()

当线程池涉及到要移除worker时候都会调用tryTerminate()，该方法主要用于判断线程池中的线程是否已经全部移除了，如果是的话则关闭线程池。

```
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 线程池处于Running状态
            // 线程池已经终止了
            // 线程池处于ShutDown状态，但是阻塞队列不为空
            if (isRunning(c) ||
                    runStateAtLeast(c, TIDYING) ||
                    (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;

            // 执行到这里，就意味着线程池要么处于STOP状态，要么处于SHUTDOWN且阻塞队列为空
            // 这时如果线程池中还存在线程，则会尝试中断线程
            if (workerCountOf(c) != 0) {
                // /线程池还有线程，但是队列没有任务了，需要中断唤醒等待任务的线程
                // （runwoker的时候首先就通过w.unlock设置线程可中断，getTask最后面的catch处理中断）
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 尝试终止线程池
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        // 线程池状态转为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
        }
    }

```

在关闭线程池的过程中，如果线程池处于STOP状态或者处于SHUDOWN状态且阻塞队列为null，则线程池会调用interruptIdleWorkers()方法中断所有线程，注意ONLY_ONE== true，表示仅中断一个线程。

interruptIdleWorkers

```
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

```

onlyOne==true仅终止一个线程，否则终止所有线程。

## 线程终止

线程池ThreadPoolExecutor提供了shutdown()和shutDownNow()用于关闭线程池。

shutdown()：按过去执行已提交任务的顺序发起一个有序的关闭，但是不接受新任务。

shutdownNow() :尝试停止所有的活动执行任务、暂停等待任务的处理，并返回等待执行的任务列表。

shutdown

```
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 推进线程状态
            advanceRunState(SHUTDOWN);
            // 中断空闲的线程
            interruptIdleWorkers();
            // 交给子类实现
            onShutdown();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

```

shutdownNow

```
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            // 中断所有线程
            interruptWorkers();
            // 返回等待执行的任务列表
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }

```

与shutdown不同，shutdownNow会调用interruptWorkers()方法中断所有线程。

```
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }

```

同时会调用drainQueue()方法返回等待执行到任务列表。

```
    private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        ArrayList<Runnable> taskList = new ArrayList<Runnable>();
        q.drainTo(taskList);
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }
```



## Executors

这节其实也不是分析 Executors 这个类，因为它仅仅是工具类，它的所有方法都是 static 的。

### FixedThreadPool

创建固定长度的线程池，每次提交任务创建一个线程，直到达到线程池的最大数量，线程池的大小不再变化。

这个线程池可以创建固定线程数的线程池。特点就是可以重用固定数量线程的线程池。它的构造源码如下：







	public static ExecutorService newFixedThreadPool(int nThreads) { 
	        return new ThreadPoolExecutor(nThreads, nThreads, 0L,
	                                      TimeUnit.MILLISECONDS, 
	                                      new LinkedBlockingQueue<Runnable>()); 
	} 






*   FixedThreadPool的corePoolSize和maxiumPoolSize都被设置为创建FixedThreadPool时指定的参数nThreads。
*   0L则表示当线程池中的线程数量操作核心线程的数量时，多余的线程将被立即停止
*   最后一个参数表示FixedThreadPool使用了无界队列LinkedBlockingQueue作为线程池的做工队列，由于是无界的，当线程池的线程数达到corePoolSize后，新任务将在无界队列中等待，因此线程池的线程数量不会超过corePoolSize，同时maxiumPoolSize也就变成了一个无效的参数，并且运行中的线程池并不会拒绝任务。

FixedThreadPool运行图如下

![](https://images2015.cnblogs.com/blog/926638/201704/926638-20170420111954915-1390955029.png)

执行过程如下：

1.如果当前工作中的线程数量少于corePool的数量，就创建新的线程来执行任务。

2.当线程池的工作中的线程数量达到了corePool，则将任务加入LinkedBlockingQueue。

3.线程执行完1中的任务后会从队列中去任务。

注意LinkedBlockingQueue是无界队列，所以可以一直添加新任务到线程池。

### SingleThreadExecutor　　

SingleThreadExecutor是使用单个worker线程的Executor。特点是使用单个工作线程执行任务。它的构造源码如下：







	public static ExecutorService newSingleThreadExecutor() {
	        return new FinalizableDelegatedExecutorService
	            (new ThreadPoolExecutor(1, 1,
	                                    0L, TimeUnit.MILLISECONDS,
	                                    new LinkedBlockingQueue<Runnable>()));
	}






SingleThreadExecutor的corePoolSize和maxiumPoolSize都被设置1。

其他参数均与FixedThreadPool相同，其运行图如下：

 ![](https://images2015.cnblogs.com/blog/926638/201704/926638-20170420112245462-2072409666.png)

执行过程如下：

1.如果当前工作中的线程数量少于corePool的数量，就创建一个新的线程来执行任务。

2.当线程池的工作中的线程数量达到了corePool，则将任务加入LinkedBlockingQueue。

3.线程执行完1中的任务后会从队列中去任务。

注意：由于在线程池中只有一个工作线程，所以任务可以按照添加顺序执行。

### CachedThreadPool

 CachedThreadPool是一个”无限“容量的线程池，它会根据需要创建新线程。特点是可以根据需要来创建新的线程执行任务，没有特定的corePool。下面是它的构造方法：







	public static ExecutorService newCachedThreadPool() {
	        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
	                                      60L, TimeUnit.SECONDS,
	                                      new SynchronousQueue<Runnable>());
	}







CachedThreadPool的corePoolSize被设置为0，即corePool为空；maximumPoolSize被设置为Integer.MAX_VALUE，即maximum是无界的。这里keepAliveTime设置为60秒，意味着空闲的线程最多可以等待任务60秒，否则将被回收。

CachedThreadPool使用没有容量的SynchronousQueue作为主线程池的工作队列，它是一个没有容量的阻塞队列。每个插入操作必须等待另一个线程的对应移除操作。这意味着，如果主线程提交任务的速度高于线程池中处理任务的速度时，CachedThreadPool会不断创建新线程。极端情况下，CachedThreadPool会因为创建过多线程而耗尽CPU资源。其运行图如下：

![](https://images2015.cnblogs.com/blog/926638/201704/926638-20170420112615102-1872791768.png)

执行过程如下：



1.首先执行SynchronousQueue.offer(Runnable task)。如果在当前的线程池中有空闲的线程正在执行SynchronousQueue.poll()，那么主线程执行的offer操作与空闲线程执行的poll操作配对成功，主线程把任务交给空闲线程执行。，execute()方法执行成功，否则执行步骤2

2.当线程池为空(初始maximumPool为空)或没有空闲线程时，配对失败，将没有线程执行SynchronousQueue.poll操作。这种情况下，线程池会创建一个新的线程执行任务。

3.在创建完新的线程以后，将会执行poll操作。当步骤2的线程执行完成后，将等待60秒，如果此时主线程提交了一个新任务，那么这个空闲线程将执行新任务，否则被回收。因此长时间不提交任务的CachedThreadPool不会占用系统资源。

SynchronousQueue是一个不存储元素阻塞队列，每次要进行offer操作时必须等待poll操作，否则不能继续添加元素。





> 具体使用案例



(1). newCachedThreadPool
创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。示例代码如下：





Java



 
	ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
	for (int i = 0; i < 10; i++) {
	final int index = i;
	try {
	Thread.sleep(index * 1000);
	} catch (InterruptedException e) {
	e.printStackTrace();
	}
	  
	cachedThreadPool.execute(new Runnable() {
	  
	@Override
	public void run() {
	System.out.println(index);
	}
	});
	}









线程池为无限大，当执行第二个任务时第一个任务已经完成，会复用执行第一个任务的线程，而不用每次新建线程。

(2). newFixedThreadPool
创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。示例代码如下：





Java



	ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
	for (int i = 0; i < 10; i++) {
	final int index = i;
	fixedThreadPool.execute(new Runnable() {
	  
	@Override
	public void run() {
	try {
	System.out.println(index);
	Thread.sleep(2000);
	} catch (InterruptedException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
	}
	}
	});
	}









因为线程池大小为3，每个任务输出index后sleep 2秒，所以每两秒打印3个数字。

定长线程池的大小最好根据系统资源进行设置。如Runtime.getRuntime().availableProcessors()。可参考[PreloadDataCache](http://www.trinea.cn/android/preloaddatacache%E6%94%AF%E6%8C%81%E9%A2%84%E5%8F%96%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BC%93%E5%AD%98%EF%BC%8C%E4%BD%BF%E7%94%A8%E7%AE%80%E5%8D%95%EF%BC%8C%E6%94%AF%E6%8C%81%E5%A4%9A%E7%A7%8D%E7%BC%93/ "PreloadDataCache支持预取的数据缓存，使用简单，支持多种缓存算法，支持不同网络类型，扩展性强")。

(3) newScheduledThreadPool
创建一个定长线程池，支持定时及周期性任务执行。延迟执行示例代码如下：





Java











	ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
	scheduledThreadPool.schedule(new Runnable() {
	  
	@Override
	public void run() {
	System.out.println("delay 3 seconds");
	}
	}, 3, TimeUnit.SECONDS);











表示延迟3秒执行。

定期执行示例代码如下：





Java










	scheduledThreadPool.scheduleAtFixedRate(new Runnable() {
	  
	@Override
	public void run() {
	System.out.println("delay 1 seconds, and excute every 3 seconds");
	}
	}, 1, 3, TimeUnit.SECONDS);











表示延迟1秒后每3秒执行一次。

ScheduledExecutorService比Timer更安全，功能更强大，后面会有一篇单独进行对比。

(4)、newSingleThreadExecutor
创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。示例代码如下：





Java



 







	ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
	for (int i = 0; i < 10; i++) {
	final int index = i;
	singleThreadExecutor.execute(new Runnable() {
	  
	@Override
	public void run() {
	try {
	System.out.println(index);
	Thread.sleep(2000);
	} catch (InterruptedException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
	}
	}
	});
	}











结果依次输出，相当于顺序执行各个任务。

现行大多数GUI程序都是单线程的。Android中单线程可用于[数据库操作](http://www.trinea.cn/android/database-performance/ "性能优化之数据库优化")，文件操作，应用批量安装，应用批量删除等不适合并发但可能IO阻塞性及影响UI线程响应的操作。

### 特别的线程池：[ScheduledThreadPoolExecutor](http://cmsblogs.com/?p=2451)

原文出处[http://cmsblogs.com/](http://cmsblogs.com/)chenssy

在上篇博客[【死磕Java并发】-----J.U.C之线程池：ThreadPoolExecutor](http://cmsblogs.com/?p=2448)已经介绍了线程池中最核心的类ThreadPoolExecutor，这篇就来看看另一个核心类ScheduledThreadPoolExecutor的实现。

## ScheduledThreadPoolExecutor解析

我们知道Timer与TimerTask虽然可以实现线程的周期和延迟调度，但是Timer与TimerTask存在一些缺陷，所以对于这种定期、周期执行任务的调度策略，我们一般都是推荐ScheduledThreadPoolExecutor来实现。下面就深入分析ScheduledThreadPoolExecutor是如何来实现线程的周期、延迟调度的。

ScheduledThreadPoolExecutor，继承ThreadPoolExecutor且实现了ScheduledExecutorService接口，它就相当于提供了“延迟”和“周期执行”功能的ThreadPoolExecutor。在JDK API中是这样定义它的：ThreadPoolExecutor，它可另行安排在给定的延迟后运行命令，或者定期执行命令。需要多个辅助线程时，或者要求 ThreadPoolExecutor 具有额外的灵活性或功能时，此类要优于 Timer。 一旦启用已延迟的任务就执行它，但是有关何时启用，启用后何时执行则没有任何实时保证。按照提交的先进先出 (FIFO) 顺序来启用那些被安排在同一执行时间的任务。

它提供了四个构造方法：

```
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                new DelayedWorkQueue());
    }

    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                new DelayedWorkQueue(), threadFactory);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                new DelayedWorkQueue(), handler);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                new DelayedWorkQueue(), threadFactory, handler);
    }

```

当然我们一般都不会直接通过其构造函数来生成一个ScheduledThreadPoolExecutor对象（例如new ScheduledThreadPoolExecutor(10)之类的），而是通过Executors类（例如Executors.newScheduledThreadPool(int);）

在ScheduledThreadPoolExecutor的构造函数中，我们发现它都是利用ThreadLocalExecutor来构造的，唯一变动的地方就在于它所使用的阻塞队列变成了DelayedWorkQueue，而不是ThreadLocalhExecutor的LinkedBlockingQueue（通过Executors产生ThreadLocalhExecutor对象）。DelayedWorkQueue为ScheduledThreadPoolExecutor中的内部类，它其实和阻塞队列DelayQueue有点儿类似。DelayQueue是可以提供延迟的阻塞队列，它只有在延迟期满时才能从中提取元素，其列头是延迟期满后保存时间最长的Delayed元素。如果延迟都还没有期满，则队列没有头部，并且 poll 将返回 null。有关于DelayQueue的更多介绍请参考这篇博客[【死磕Java并发】-----J.U.C之阻塞队列：DelayQueue](http://cmsblogs.com/?p=2413)。所以DelayedWorkQueue中的任务必然是按照延迟时间从短到长来进行排序的。下面我们再深入分析DelayedWorkQueue，这里留一个引子。

ScheduledThreadPoolExecutor提供了如下四个方法，也就是四个调度器：

1.  schedule(Callable callable, long delay, TimeUnit unit) :创建并执行在给定延迟后启用的 ScheduledFuture。
2.  schedule(Runnable command, long delay, TimeUnit unit) :创建并执行在给定延迟后启用的一次性操作。
3.  scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) :创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。
4.  scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) :创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。

第一、二个方法差不多，都是一次性操作，只不过参数一个是Callable，一个是Runnable。稍微分析下第三（scheduleAtFixedRate）、四个（scheduleWithFixedDelay）方法，加入initialDelay = 5，period/delay = 3，unit为秒。如果每个线程都是都运行非常良好不存在延迟的问题，那么这两个方法线程运行周期是5、8、11、14、17.......，但是如果存在延迟呢？比如第三个线程用了5秒钟，那么这两个方法的处理策略是怎样的？第三个方法（scheduleAtFixedRate）是周期固定，也就说它是不会受到这个延迟的影响的，每个线程的调度周期在初始化时就已经绝对了，是什么时候调度就是什么时候调度，它不会因为上一个线程的调度失效延迟而受到影响。但是第四个方法（scheduleWithFixedDelay），则不一样，它是每个线程的调度间隔固定，也就是说第一个线程与第二线程之间间隔delay，第二个与第三个间隔delay，以此类推。如果第二线程推迟了那么后面所有的线程调度都会推迟，例如，上面第二线程推迟了2秒，那么第三个就不再是11秒执行了，而是13秒执行。

查看着四个方法的源码，会发现其实他们的处理逻辑都差不多，所以我们就挑scheduleWithFixedDelay方法来分析，如下：

```
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

```

scheduleWithFixedDelay方法处理的逻辑如下：

1.  校验，如果参数不合法则抛出异常
2.  构造一个task，该task为ScheduledFutureTask
3.  调用delayedExecute()方法做后续相关处理

这段代码涉及两个类ScheduledFutureTask和RunnableScheduledFuture，其中RunnableScheduledFuture不用多说，他继承RunnableFuture和ScheduledFuture两个接口，除了具备RunnableFuture和ScheduledFuture两类特性外，它还定义了一个方法isPeriodic() ，该方法用于判断执行的任务是否为定期任务，如果是则返回true。而ScheduledFutureTask作为ScheduledThreadPoolExecutor的内部类，它扮演着极其重要的作用，因为它的作用则是负责ScheduledThreadPoolExecutor中任务的调度。

ScheduledFutureTask内部继承FutureTask，实现RunnableScheduledFuture接口，它内部定义了三个比较重要的变量

```
        /** 任务被添加到ScheduledThreadPoolExecutor中的序号 */
        private final long sequenceNumber;

        /** 任务要执行的具体时间 */
        private long time;

        /**  任务的间隔周期 /
        private final long period;

```

这三个变量与任务的执行有着非常密切的关系，什么关系？先看ScheduledFutureTask的几个构造函数和核心方法：

```
        ScheduledFutureTask(Runnable r, V result, long ns) {
            super(r, result);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        ScheduledFutureTask(Runnable r, V result, long ns, long period) {
            super(r, result);
            this.time = ns;
            this.period = period;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        ScheduledFutureTask(Callable<V> callable, long ns) {
            super(callable);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        ScheduledFutureTask(Callable<V> callable, long ns) {
            super(callable);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

```

ScheduledFutureTask 提供了四个构造方法，这些构造方法与上面三个参数是不是一一对应了？这些参数有什么用，如何用，则要看ScheduledFutureTask在那些方法使用了该方法，在ScheduledFutureTask中有一个compareTo()方法：

```
    public int compareTo(Delayed other) {
        if (other == this) // compare zero if same object
            return 0;
        if (other instanceof ScheduledFutureTask) {
            ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
            long diff = time - x.time;
            if (diff < 0)
                return -1;
            else if (diff > 0)
                return 1;
            else if (sequenceNumber < x.sequenceNumber)
                return -1;
            else
                return 1;
        }
        long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
        return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
    }

```

相信各位都知道该方法是干嘛用的，提供一个排序算法，该算法规则是：首先按照time排序，time小的排在前面，大的排在后面，如果time相同，则使用sequenceNumber排序，小的排在前面，大的排在后面。那么为什么在这个类里面提供compareTo()方法呢？在前面就介绍过ScheduledThreadPoolExecutor在构造方法中提供的是DelayedWorkQueue()队列中，也就是说ScheduledThreadPoolExecutor是把任务添加到DelayedWorkQueue中的，而DelayedWorkQueue则是类似于DelayQueue，内部维护着一个以时间为先后顺序的队列，所以compareTo()方法使用与DelayedWorkQueue队列对其元素ScheduledThreadPoolExecutor task进行排序的算法。

排序已经解决了，那么ScheduledThreadPoolExecutor 是如何对task任务进行调度和延迟的呢？任何线程的执行，都是通过run()方法执行，ScheduledThreadPoolExecutor 的run()方法如下：

```
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            else if (!periodic)
                ScheduledFutureTask.super.run();
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }

```

1.  调用isPeriodic()获取该线程是否为周期性任务标志，然后调用canRunInCurrentRunState()方法判断该线程是否可以执行，如果不可以执行则调用cancel()取消任务。
2.  如果当线程已经到达了执行点，则调用run()方法执行task，该run()方法是在FutureTask中定义的。
3.  否则调用runAndReset()方法运行并充值，调用setNextRunTime()方法计算任务下次的执行时间，重新把任务添加到队列中，让该任务可以重复执行。

isPeriodic()

该方法用于判断指定的任务是否为定期任务。

```
        public boolean isPeriodic() {
            return period != 0;
        }

```

canRunInCurrentRunState()判断任务是否可以取消，cancel()取消任务，这两个方法比较简单，而run()执行任务，runAndReset()运行并重置状态，牵涉比较广，我们放在FutureTask后面介绍。所以重点介绍setNextRunTime()和reExecutePeriodic()这两个涉及到延迟的方法。

setNextRunTime()

setNextRunTime()方法用于重新计算任务的下次执行时间。如下：

```
        private void setNextRunTime() {
            long p = period;
            if (p > 0)
                time += p;
            else
                time = triggerTime(-p);
        }

```

该方法定义很简单，p > 0 ,time += p ，否则调用triggerTime()方法重新计算time：

```
    long triggerTime(long delay) {
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
    }

```

reExecutePeriodic

```
    void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        if (canRunInCurrentRunState(true)) {
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }

```

reExecutePeriodic重要的是调用super.getQueue().add(task);将任务task加入的队列DelayedWorkQueue中。ensurePrestart()在[【死磕Java并发】-----J.U.C之线程池：ThreadPoolExecutor](http://cmsblogs.com/?p=2448)已经做了详细介绍。

到这里ScheduledFutureTask已经介绍完了，ScheduledFutureTask在ScheduledThreadPoolExecutor扮演作用的重要性不言而喻。其实ScheduledThreadPoolExecutor的实现不是很复杂，因为有FutureTask和ThreadPoolExecutor的支撑，其实现就显得不是那么难了。

## 总结

我一向不喜欢写总结，因为我把所有需要表达的都写在正文中了，写小篇幅的总结并不能真正将话说清楚，本文的总结部分为准备面试的读者而写，希望能帮到面试者或者没有足够的时间看完全文的读者。

1.  java 线程池有哪些关键属性？

    > corePoolSize，maximumPoolSize，workQueue，keepAliveTime，rejectedExecutionHandler
    > 
    > corePoolSize 到 maximumPoolSize 之间的线程会被回收，当然 corePoolSize 的线程也可以通过设置而得到回收（allowCoreThreadTimeOut(true)）。
    > 
    > workQueue 用于存放任务，添加任务的时候，如果当前线程数超过了 corePoolSize，那么往该队列中插入任务，线程池中的线程会负责到队列中拉取任务。
    > 
    > keepAliveTime 用于设置空闲时间，如果线程数超出了 corePoolSize，并且有些线程的空闲时间超过了这个值，会执行关闭这些线程的操作
    > 
    > rejectedExecutionHandler 用于处理当线程池不能执行此任务时的情况，默认有抛出 RejectedExecutionException 异常、忽略任务、使用提交任务的线程来执行此任务和将队列中等待最久的任务删除，然后提交此任务这四种策略，默认为抛出异常。

2.  说说线程池中的线程创建时机？

    > 1.  如果当前线程数少于 corePoolSize，那么提交任务的时候创建一个新的线程，并由这个线程执行这个任务；
    > 2.  如果当前线程数已经达到 corePoolSize，那么将提交的任务添加到队列中，等待线程池中的线程去队列中取任务；
    > 3.  如果队列已满，那么创建新的线程来执行任务，需要保证池中的线程数不会超过 maximumPoolSize，如果此时线程数超过了 maximumPoolSize，那么执行拒绝策略。

    * 注意：如果将队列设置为无界队列，那么线程数达到 corePoolSize 后，其实线程数就不会再增长了。

3.  Executors.newFixedThreadPool(…) 和 Executors.newCachedThreadPool() 构造出来的线程池有什么差别？

    > 细说太长，往上滑一点点，在 Executors 的小节进行了详尽的描述。

4.  任务执行过程中发生异常怎么处理？

    > 如果某个任务执行出现异常，那么执行任务的线程会被关闭，而不是继续接收其他任务。然后会启动一个新的线程来代替它。

5.  什么时候会执行拒绝策略？

    > 1.  workers 的数量达到了 corePoolSize（任务此时需要进入任务队列），任务入队成功，与此同时线程池被关闭了，而且关闭线程池并没有将这个任务出队，那么执行拒绝策略。这里说的是非常边界的问题，入队和关闭线程池并发执行，读者仔细看看 execute 方法是怎么进到第一个 reject(command) 里面的。
    > 2.  workers 的数量大于等于 corePoolSize，将任务加入到任务队列，可是队列满了，任务入队失败，那么准备开启新的线程，可是线程数已经达到 maximumPoolSize，那么执行拒绝策略。