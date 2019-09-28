---
title: Java并发指南15：Fork join并发框架与工作窃取算法剖析
date: 2018-05-24 22:47:05
tags:
    - Java并发
categories:
	- 后端
	- Java并发
---
本文首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Java并发指南 
https://blog.csdn.net/column/details/21961.html

相关代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->

转载自_[并发编程网 – ifeve.com](http://ifeve.com/)

## 什么是Fork/Join框架

Fork/Join框架是Java7提供了的一个用于并行执行任务的框架， 是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。

我们再通过Fork和Join这两个单词来理解下Fork/Join框架，Fork就是把一个大任务切分为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大任务的结果。比如计算1+2+。。＋10000，可以分割成10个子任务，每个子任务分别对1000个数进行求和，最终汇总这10个子任务的结果。Fork/Join的运行流程图如下：

![](http://infoqstatic.com/resource/articles/fork-join-introduction/zh/resources/21.png)


## 工作窃取算法

工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。工作窃取的运行流程图如下：

![fj](http://ifeve.com/wp-content/uploads/2013/12/fj.png)

那么为什么需要使用工作窃取算法呢？假如我们需要做一个比较大的任务，我们可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，于是把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应，比如A线程负责处理A队列里的任务。但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

## Fork/Join框架的介绍

我们已经很清楚Fork/Join框架的需求了，那么我们可以思考一下，如果让我们来设计一个Fork/Join框架，该如何设计？这个思考有助于你理解Fork/Join框架的设计。

第一步分割任务。首先我们需要有一个fork类来把大任务分割成子任务，有可能子任务还是很大，所以还需要不停的分割，直到分割出的子任务足够小。

第二步执行任务并合并结果。分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

Fork/Join使用两个类来完成以上两件事情：

*   ForkJoinTask：我们要使用ForkJoin框架，必须首先创建一个ForkJoin任务。它提供在任务中执行fork()和join()操作的机制，通常情况下我们不需要直接继承ForkJoinTask类，而只需要继承它的子类，Fork/Join框架提供了以下两个子类：
    *   RecursiveAction：用于没有返回结果的任务。
    *   RecursiveTask ：用于有返回结果的任务。
*   ForkJoinPool ：ForkJoinTask需要通过ForkJoinPool来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。

## 使用Fork/Join框架

让我们通过一个简单的需求来使用下Fork／Join框架，需求是：计算1+2+3+4的结果。

使用Fork／Join框架首先要考虑到的是如何分割任务，如果我们希望每个子任务最多执行两个数的相加，那么我们设置分割的阈值是2，由于是4个数字相加，所以Fork／Join框架会把这个任务fork成两个子任务，子任务一负责计算1+2，子任务二负责计算3+4，然后再join两个子任务的结果。

因为是有结果的任务，所以必须继承RecursiveTask，实现代码如下：

	001	packagefj;
	002	 
	003	importjava.util.concurrent.ExecutionException;
	004	 
	005	importjava.util.concurrent.ForkJoinPool;
	006	 
	007	importjava.util.concurrent.Future;
	008	 
	009	importjava.util.concurrent.RecursiveTask;
	010	 
	011	publicclassCountTaskextendsRecursiveTask {
	012	 
	013	       privatestaticfinalintTHRESHOLD= 2;//阈值
	014	 
	015	       privateintstart;
	016	 
	017	       privateintend;
	018	 
	019	       publicCountTask(intstart,intend) {
	020	 
	021	                   this.start= start;
	022	 
	023	                   this.end= end;
	024	 
	025	        }
	026	 
	027	       @Override
	028	 
	029	       protectedInteger compute() {
	030	 
	031	                   intsum = 0;
	032	 
	033	                   //如果任务足够小就计算任务
	034	 
	035	                   booleancanCompute = (end-start) <=THRESHOLD;
	036	 
	037	                   if(canCompute) {
	038	 
	039	                              for(inti =start; i <=end; i++) {
	040	 
	041	                                           sum += i;
	042	 
	043	                               }
	044	 
	045	                    }else{
	046	 
	047	                              //如果任务大于阀值，就分裂成两个子任务计算
	048	 
	049	                              intmiddle = (start+end) / 2;
	050	 
	051	                               CountTask leftTask =newCountTask(start, middle);
	052	 
	053	                               CountTask rightTask =newCountTask(middle + 1,end);
	054	 
	055	                              //执行子任务
	056	 
	057	                               leftTask.fork();
	058	 
	059	                               rightTask.fork();
	060	 
	061	                              //等待子任务执行完，并得到其结果
	062	 
	063	                              intleftResult=leftTask.join();
	064	 
	065	                              intrightResult=rightTask.join();
	066	 
	067	                              //合并子任务
	068	 
	069	                               sum = leftResult  + rightResult;
	070	 
	071	                    }
	072	 
	073	                   returnsum;
	074	 
	075	        }
	076	 
	077	       publicstaticvoidmain(String[] args) {
	078	 
	079	                    ForkJoinPool forkJoinPool =newForkJoinPool();
	080	 
	081	                   //生成一个计算任务，负责计算1+2+3+4
	082	 
	083	                    CountTask task =newCountTask(1, 4);
	084	 
	085	                   //执行一个任务
	086	 
	087	                    Future result = forkJoinPool.submit(task);
	088	 
	089	                   try{
	090	 
	091	                               System.out.println(result.get());
	092	 
	093	                    }catch(InterruptedException e) {
	094	 
	095	                    }catch(ExecutionException e) {
	096	 
	097	                    }
	098	 
	099	        }
	100	 
	101	}







通过这个例子让我们再来进一步了解ForkJoinTask，ForkJoinTask与一般的任务的主要区别在于它需要实现compute方法，在这个方法里，首先需要判断任务是否足够小，如果足够小就直接执行任务。如果不足够小，就必须分割成两个子任务，每个子任务在调用fork方法时，又会进入compute方法，看看当前子任务是否需要继续分割成孙任务，如果不需要继续分割，则执行当前子任务并返回结果。使用join方法会等待子任务执行完并得到其结果。

## Fork/Join框架的异常处理

ForkJoinTask在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以ForkJoinTask提供了isCompletedAbnormally()方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过ForkJoinTask的getException方法获取异常。使用如下代码：

<pre>if(task.isCompletedAbnormally())
{
    System.out.println(task.getException());
}</pre>

getException方法返回Throwable对象，如果任务被取消了则返回CancellationException。如果任务没有完成或者没有抛出异常则返回null。

## Fork/Join框架的实现原理

ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，ForkJoinTask数组负责存放程序提交给ForkJoinPool的任务，而ForkJoinWorkerThread数组负责执行这些任务。

ForkJoinTask的fork方法实现原理。当我们调用ForkJoinTask的fork方法时，程序会调用ForkJoinWorkerThread的pushTask方法异步的执行这个任务，然后立即返回结果。代码如下：






	1	public final ForkJoinTask fork() {
	2	        ((ForkJoinWorkerThread) Thread.currentThread())
	3	            .pushTask(this);
	4	        return this;
	5	}




pushTask方法把当前任务存放在ForkJoinTask 数组queue里。然后再调用ForkJoinPool的signalWork()方法唤醒或创建一个工作线程来执行任务。代码如下：






	01	final void pushTask(ForkJoinTask t) {
	02	        ForkJoinTask[] q; int s, m;
	03	        if ((q = queue) != null) {    // ignore if queue removed
	04	            long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
	05	            UNSAFE.putOrderedObject(q, u, t);
	06	            queueTop = s + 1;         // or use putOrderedInt
	07	            if ((s -= queueBase) <= 2)
	08	                pool.signalWork();
	09	    else if (s == m)
	10	                growQueue();
	11	        }
	12	    }






ForkJoinTask的join方法实现原理。Join方法的主要作用是阻塞当前线程并等待获取结果。让我们一起看看ForkJoinTask的join方法的实现，代码如下：






	public final V join() {
	02	        if (doJoin() != NORMAL)
	03	            return reportResult();
	04	        else
	05	            return getRawResult();
	06	}
	07	private V reportResult() {
	08	        int s; Throwable ex;
	09	        if ((s = status) == CANCELLED)
	10	            throw new CancellationException();
	11	if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
	12	            UNSAFE.throwException(ex);
	13	        return getRawResult();
	14	}







首先，它调用了doJoin()方法，通过doJoin()方法得到当前任务的状态来判断返回什么结果，任务状态有四种：已完成（NORMAL），被取消（CANCELLED），信号（SIGNAL）和出现异常（EXCEPTIONAL）。

*   如果任务状态是已完成，则直接返回任务结果。
*   如果任务状态是被取消，则直接抛出CancellationException。
*   如果任务状态是抛出异常，则直接抛出对应的异常。

让我们再来分析下doJoin()方法的实现代码：







01	private int doJoin() {
02	        Thread t; ForkJoinWorkerThread w; int s; booleancompleted;
03	        if ((t = Thread.currentThread()) instanceofForkJoinWorkerThread) {
04	            if ((s = status) < 0)
05	 return s;
06	            if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) {
07	                try {
08	                    completed = exec();
09	                } catch (Throwable rex) {
10	                    return setExceptionalCompletion(rex);
11	                }
12	                if (completed)
13	                    return setCompletion(NORMAL);
14	            }
15	            return w.joinTask(this);
16	        }
17	        else
18	            return externalAwaitDone();
19	    }






在doJoin()方法里，首先通过查看任务的状态，看任务是否已经执行完了，如果执行完了，则直接返回任务状态，如果没有执行完，则从任务数组里取出任务并执行。如果任务顺利执行完成了，则设置任务状态为NORMAL，如果出现异常，则纪录异常，并将任务状态设置为EXCEPTIONAL。

## Fork/Join源码剖析与算法解析

我们在大学算法课本上，学过的一种基本算法就是：分治。其基本思路就是：把一个大的任务分成若干个子任务，这些子任务分别计算，最后再Merge出最终结果。这个过程通常都会用到递归。

而Fork/Join其实就是一种利用多线程来实现“分治算法”的并行框架。

另外一方面，可以把Fori/Join看作一个单机版的Map/Reduce，只不过这里的并行不是多台机器并行计算，而是多个线程并行计算。

下面看2个简单例子：

例子1： 快排 
我们都知道，快排有2个步骤： 
第1步，拿数组的第1个元素，把元素划分成2半，左边的比该元素小，右边的比该元素大； 
第2步，对左右的2个子数组，分别排序。

可以看出，这里左右2个子数组，可以相互独立的，并行计算。因此可以利用ForkJoin框架， 代码如下：

```
//定义一个Task，基础自RecursiveAction，实现其compute方法
class SortTask extends RecursiveAction {
    final long[] array;
    final int lo;
    final int hi;
    private int THRESHOLD = 0; //For demo only

    public SortTask(long[] array) {
        this.array = array;
        this.lo = 0;
        this.hi = array.length - 1;
    }

    public SortTask(long[] array, int lo, int hi) {
        this.array = array;
        this.lo = lo;
        this.hi = hi;
    }

    protected void compute() {
        if (hi - lo < THRESHOLD)
            sequentiallySort(array, lo, hi);
        else {
            int pivot = partition(array, lo, hi);  //划分
            coInvoke(new SortTask(array, lo, pivot - 1), new SortTask(array,
                    pivot + 1, hi));  //递归调，左右2个子数组
        }
    }

    private int partition(long[] array, int lo, int hi) {
        long x = array[hi];
        int i = lo - 1;
        for (int j = lo; j < hi; j++) {
            if (array[j] <= x) {
                i++;
                swap(array, i, j);
            }
        }
        swap(array, i + 1, hi);
        return i + 1;
    }

    private void swap(long[] array, int i, int j) {
        if (i != j) {
            long temp = array[i];
            array[i] = array[j];
            array[j] = temp;
        }
    }

    private void sequentiallySort(long[] array, int lo, int hi) {
        Arrays.sort(array, lo, hi + 1);
    }
}

//测试函数
    public void testSort() throws Exception {
        ForkJoinTask sort = new SortTask(array);   //1个任务
        ForkJoinPool fjpool = new ForkJoinPool();  //1个ForkJoinPool
        fjpool.submit(sort); //提交任务
        fjpool.shutdown(); //结束。ForkJoinPool内部会开多个线程，并行上面的子任务

        fjpool.awaitTermination(30, TimeUnit.SECONDS);
    }
```

例子2： 求1到n个数的和

```
//定义一个Task，基础自RecursiveTask，实现其commpute方法
public class SumTask extends RecursiveTask<Long>{  
    private static final int THRESHOLD = 10;  

    private long start;  
    private long end;  

    public SumTask(long n) {  
        this(1,n);  
    }  

    private SumTask(long start, long end) {  
        this.start = start;  
        this.end = end;  
    }  

    @Override  //有返回值
    protected Long compute() {  
        long sum = 0;  
        if((end - start) <= THRESHOLD){  
            for(long l = start; l <= end; l++){  
                sum += l;  
            }  
        }else{  
            long mid = (start + end) >>> 1;  
            SumTask left = new SumTask(start, mid);   //分治，递归
            SumTask right = new SumTask(mid + 1, end);  
            left.fork();  
            right.fork();  
            sum = left.join() + right.join();  
        }  
        return sum;  
    }  
    private static final long serialVersionUID = 1L;  
}  

//测试函数
    public void testSum() throws Exception {
        SumTask sum = new SumTask(100);   //1个任务
        ForkJoinPool fjpool = new ForkJoinPool();  //1个ForkJoinPool
        Future<Long> future = fjpool.submit(sum); //提交任务
        Long r = future.get(); //获取返回值
        fjpool.shutdown(); 

    }
```

与ThreadPool的区别

通过上面例子，我们可以看出，它在使用上，和ThreadPool有共同的地方，也有区别点： 
（1） ThreadPool只有“外部任务”，也就是调用者放到队列里的任务。 ForkJoinPool有“外部任务”，还有“内部任务”，也就是任务自身在执行过程中，分裂出”子任务“，递归，再次放入队列。 
（2）ForkJoinPool里面的任务通常有2类，RecusiveAction/RecusiveTask，这2个都是继承自FutureTask。在使用的时候，重写其compute算法。

工作窃取算法

上面提到，ForkJoinPool里有”外部任务“，也有“内部任务”。其中外部任务，是放在ForkJoinPool的全局队列里面，而每个Worker线程，也有一个自己的队列，用于存放内部任务。

窃取的基本思路就是：当worker自己的任务队列里面没有任务时，就去scan别的线程的队列，把别人的任务拿过来执行。

```
//ForkJoinPool的成员变量
ForkJoinWorkerThread[] workers;  //worker thread集合
private ForkJoinTask<?>[] submissionQueue; //外部任务队列
private final ReentrantLock submissionLock; 

//ForkJoinWorkerThread的成员变量
ForkJoinTask<?>[] queue;   //每个worker线程自己的内部任务队列

//提交任务
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {  
    if (task == null)  
        throw new NullPointerException();  
    forkOrSubmit(task);  
    return task;  
} 

private <T> void forkOrSubmit(ForkJoinTask<T> task) {  
    ForkJoinWorkerThread w;  
    Thread t = Thread.currentThread();  
    if (shutdown)  
        throw new RejectedExecutionException();  
    if ((t instanceof ForkJoinWorkerThread) &&   //如果当前是worker线程提交的任务，也就是worker执行过程中，分裂出来的子任务，放入worker自己的内部任务队列
        (w = (ForkJoinWorkerThread)t).pool == this)  
        w.pushTask(task);  
    else  
        addSubmission(task);  //外部任务，放入pool的全局队列
}   

//worker的run方法
public void run() {  
    Throwable exception = null;  
    try {  
        onStart();  
        pool.work(this);  
    } catch (Throwable ex) {  
        exception = ex;  
    } finally {  
        onTermination(exception);  
    }  
}  

final void work(ForkJoinWorkerThread w) {  
    boolean swept = false;                // true on empty scans  
    long c;  
    while (!w.terminate && (int)(c = ctl) >= 0) {  
        int a;                            // active count  
        if (!swept && (a = (int)(c >> AC_SHIFT)) <= 0)  
            swept = scan(w, a);   //核心代码都在这个scan函数里面
        else if (tryAwaitWork(w, c))  
            swept = false;  
    }  
}  

//scan的基本思路：从别人的任务队列里面抢，没有，再到pool的全局的任务队列里面去取。
private boolean scan(ForkJoinWorkerThread w, int a) {  
    int g = scanGuard;   

    int m = (parallelism == 1 - a && blockedCount == 0) ? 0 : g & SMASK;  
    ForkJoinWorkerThread[] ws = workers;  
    if (ws == null || ws.length <= m)         // 过期检测  
        return false;  

    for (int r = w.seed, k = r, j = -(m + m); j <= m + m; ++j) {  
        ForkJoinTask<?> t; ForkJoinTask<?>[] q; int b, i;  
        //随机选出一个牺牲者(工作线程)。  
        ForkJoinWorkerThread v = ws[k & m];  
        //一系列检查...  
        if (v != null && (b = v.queueBase) != v.queueTop &&  
            (q = v.queue) != null && (i = (q.length - 1) & b) >= 0) {  
            //如果这个牺牲者的任务队列中还有任务，尝试窃取这个任务。  
            long u = (i << ASHIFT) + ABASE;  
            if ((t = q[i]) != null && v.queueBase == b &&  
                UNSAFE.compareAndSwapObject(q, u, t, null)) {  
                //窃取成功后，调整queueBase  
                int d = (v.queueBase = b + 1) - v.queueTop;  
                //将牺牲者的stealHint设置为当前工作线程在pool中的下标。  
                v.stealHint = w.poolIndex;  
                if (d != 0)  
                    signalWork();             // 如果牺牲者的任务队列还有任务，继续唤醒(或创建)线程。  
                w.execTask(t); //执行窃取的任务。  
            }  
            //计算出下一个随机种子。  
            r ^= r << 13; r ^= r >>> 17; w.seed = r ^ (r << 5);  
            return false;                     // 返回false，表示不是一个空扫描。  
        }  
        //前2*m次，随机扫描。  
        else if (j < 0) {                     // xorshift  
            r ^= r << 13; r ^= r >>> 17; k = r ^= r << 5;  
        }  
        //后2*m次，顺序扫描。  
        else  
            ++k;  
    }  
    if (scanGuard != g)                       // staleness check  
        return false;  
    else {                                     
        //如果扫描完毕后没找到可窃取的任务，那么从Pool的提交任务队列中取一个任务来执行。  
        ForkJoinTask<?> t; ForkJoinTask<?>[] q; int b, i;  
        if ((b = queueBase) != queueTop &&  
            (q = submissionQueue) != null &&  
            (i = (q.length - 1) & b) >= 0) {  
            long u = (i << ASHIFT) + ABASE;  
            if ((t = q[i]) != null && queueBase == b &&  
                UNSAFE.compareAndSwapObject(q, u, t, null)) {  
                queueBase = b + 1;  
                w.execTask(t);  
            }  
            return false;  
        }  
        return true;                         // 如果所有的队列(工作线程的任务队列和pool的任务队列)都是空的，返回true。  
    }  
}  
```

关于ForkJoinPool/FutureTask，本文只是分析了其基本使用原理。还有很多实现细节，留待读者自己去分析。