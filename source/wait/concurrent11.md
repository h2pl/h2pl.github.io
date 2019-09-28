---
title: Java并发指南11：解读 Java阻塞队列 BlockingQueue
date: 2018-05-22 22:46:41
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

解读 Java 并发队列 BlockingQueue

转自：https://javadoop.com/post/java-concurrent-queue

最近得空，想写篇文章好好说说 java 线程池问题，我相信很多人都一知半解的，包括我自己在仔仔细细看源码之前，也有许多的不解，甚至有些地方我一直都没有理解到位。

说到线程池实现，那么就不得不涉及到各种 BlockingQueue 的实现，那么我想就 BlockingQueue 的问题和大家分享分享我了解的一些知识。

本文没有像之前分析 AQS 那样一行一行源码分析了，不过还是把其中最重要和最难理解的代码说了一遍，所以不免篇幅略长。本文涉及到比较多的 Doug Lea 对 BlockingQueue 的设计思想，希望有心的读者真的可以有一些收获，我觉得自己还是写了一些干货的。

本文直接参考 Doug Lea 写的 Java doc 和注释，这也是我们在学习 java 并发包时最好的材料了。希望大家能有所思、有所悟，学习 Doug Lea 的代码风格，并将其优雅、严谨的作风应用到我们写的每一行代码中。

目录



*   阻塞队列概览
*   Java中的阻塞队列
*   [BlockingQueue源码分析](https://javadoop.com/post/java-concurrent-queue#BlockingQueue)

*   [        BlockingQueue 实现之 ArrayBlockingQueue](https://javadoop.com/post/java-concurrent-queue#BlockingQueue%20%E5%AE%9E%E7%8E%B0%E4%B9%8B%20ArrayBlockingQueue)
*   [        BlockingQueue 实现之 LinkedBlockingQueue](https://javadoop.com/post/java-concurrent-queue#BlockingQueue%20%E5%AE%9E%E7%8E%B0%E4%B9%8B%20LinkedBlockingQueue)
*   [        BlockingQueue 实现之 SynchronousQueue](https://javadoop.com/post/java-concurrent-queue#BlockingQueue%20%E5%AE%9E%E7%8E%B0%E4%B9%8B%20SynchronousQueue)
*   [        BlockingQueue 实现之 PriorityBlockingQueue](https://javadoop.com/post/java-concurrent-queue#BlockingQueue%20%E5%AE%9E%E7%8E%B0%E4%B9%8B%20PriorityBlockingQueue)
*   [总结](https://javadoop.com/post/java-concurrent-queue#%E6%80%BB%E7%BB%93)



## 阻塞队列概览

### 什么是阻塞队列？

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

阻塞队列提供了四种处理方法:

| 方法\处理方式 | 抛出异常 | 返回特殊值 | 一直阻塞 | 超时退出 |
| --- | --- | --- | --- | --- |
| 插入方法 | add(e) | offer(e) | put(e) | offer(e,time,unit) |
| 移除方法 | remove() | poll() | take() | poll(time,unit) |
| 检查方法 | element() | peek() | 不可用 | 不可用 |

*   抛出异常：是指当阻塞队列满时候，再往队列里插入元素，会抛出IllegalStateException(“Queue full”)异常。当队列为空时，从队列里获取元素时会抛出NoSuchElementException异常 。
*   返回特殊值：插入方法会返回是否成功，成功则返回true。移除方法，则是从队列里拿出一个元素，如果没有则返回null
*   一直阻塞：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到拿到数据，或者响应中断退出。当队列空时，消费者线程试图从队列里take元素，队列也会阻塞消费者线程，直到队列可用。
*   超时退出：当阻塞队列满时，队列会阻塞生产者线程一段时间，如果超过一定的时间，生产者线程就会退出。

### Java里的阻塞队列

JDK7提供了7个阻塞队列。分别是

*   ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。
*   LinkedBlockingQueue ：一个由链表结构组成的有界阻塞队列。
*   PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。
*   DelayQueue：一个使用优先级队列实现的无界阻塞队列。
*   SynchronousQueue：一个不存储元素的阻塞队列。
*   LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
*   LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

 ArrayBlockingQueue

ArrayBlockingQueue是一个用数组实现的有界阻塞队列。此队列按照先进先出（FIFO）的原则对元素进行排序。默认情况下不保证访问者公平的访问队列，所谓公平访问队列是指阻塞的所有生产者线程或消费者线程，当队列可用时，可以按照阻塞的先后顺序访问队列，即先阻塞的生产者线程，可以先往队列里插入元素，先阻塞的消费者线程，可以先从队列里获取元素。通常情况下为了保证公平性会降低吞吐量。

 LinkedBlockingQueue

LinkedBlockingQueue是一个用链表实现的有界阻塞队列。此队列的默认和最大长度为Integer.MAX_VALUE。此队列按照先进先出的原则对元素进行排序。

 PriorityBlockingQueue

PriorityBlockingQueue是一个支持优先级的无界队列。默认情况下元素采取自然顺序排列，也可以通过比较器comparator来指定元素的排序规则。元素按照升序排列。

 DelayQueue

DelayQueue是一个支持延时获取元素的无界阻塞队列。队列使用PriorityQueue来实现。队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。我们可以将DelayQueue运用在以下应用场景：

*   缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
*   定时任务调度。使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，从比如TimerQueue就是使用DelayQueue实现的。

阻塞队列源码分析：

## BlockingQueue

首先，最基本的来说， BlockingQueue 是一个先进先出的队列（Queue），为什么说是阻塞（Blocking）的呢？是因为 BlockingQueue 支持当获取队列元素但是队列为空时，会阻塞等待队列中有元素再返回；也支持添加元素时，如果队列已满，那么等到队列可以放入新元素时再放入。

BlockingQueue 是一个接口，继承自 Queue，所以其实现类也可以作为 Queue 的实现来使用，而 Queue 又继承自 Collection 接口。

BlockingQueue 对插入操作、移除操作、获取元素操作提供了四种不同的方法用于不同的场景中使用：1、抛出异常；2、返回特殊值（null 或 true/false，取决于具体的操作）；3、阻塞等待此操作，直到这个操作成功；4、阻塞等待此操作，直到成功或者超时指定时间。总结如下：

|   | _Throws exception_ | _Special value_ | _Blocks_ | _Times out_ |
| --- | --- | --- | --- | --- |
| Insert | add(e) | offer(e) | put(e) | offer(e, time, unit) |
| Remove | remove() | poll() | take() | poll(time, unit) |
| Examine | element() | peek() | _not applicable_ | _not applicable_ |

BlockingQueue 的各个实现都遵循了这些规则，当然我们也不用死记这个表格，知道有这么回事，然后写代码的时候根据自己的需要去看方法的注释来选取合适的方法即可。

> 对于 BlockingQueue，我们的关注点应该在 put(e) 和 take() 这两个方法，因为这两个方法是带阻塞的。

BlockingQueue 不接受 null 值的插入，相应的方法在碰到 null 的插入时会抛出 NullPointerException 异常。null 值在这里通常用于作为特殊值返回（表格中的第三列），代表 poll 失败。所以，如果允许插入 null 值的话，那获取的时候，就不能很好地用 null 来判断到底是代表失败，还是获取的值就是 null 值。

一个 BlockingQueue 可能是有界的，如果在插入的时候，发现队列满了，那么 put 操作将会阻塞。通常，在这里我们说的无界队列也不是说真正的无界，而是它的容量是 Integer.MAX_VALUE（21亿多）。

BlockingQueue 是设计用来实现生产者-消费者队列的，当然，你也可以将它当做普通的 Collection 来用，前面说了，它实现了 java.util.Collection 接口。例如，我们可以用 remove(x) 来删除任意一个元素，但是，这类操作通常并不高效，所以尽量只在少数的场合使用，比如一条消息已经入队，但是需要做取消操作的时候。

BlockingQueue 的实现都是线程安全的，但是批量的集合操作如 `addAll`, `containsAll`, `retainAll` 和 `removeAll` 不一定是原子操作。如 addAll(c) 有可能在添加了一些元素后中途抛出异常，此时 BlockingQueue 中已经添加了部分元素，这个是允许的，取决于具体的实现。

BlockingQueue 不支持 close 或 shutdown 等关闭操作，因为开发者可能希望不会有新的元素添加进去，此特性取决于具体的实现，不做强制约束。

最后，BlockingQueue 在生产者-消费者的场景中，是支持多消费者和多生产者的，说的其实就是线程安全问题。

相信上面说的每一句都很清楚了，BlockingQueue 是一个比较简单的线程安全容器，下面我会分析其具体的在 JDK 中的实现，这里又到了 Doug Lea 表演时间了。



## ArrayBlockingQueue

ArrayBlockingQueue 是 BlockingQueue 接口的有界队列实现类，底层采用数组来实现。

其并发控制采用可重入锁来控制，不管是插入操作还是读取操作，都需要获取到锁才能进行操作。

如果读者看过我之前写的《[一行一行源码分析清楚 AbstractQueuedSynchronizer（二）](https://javadoop.com/post/AbstractQueuedSynchronizer-2)》 的关于 Condition 的文章的话，那么你一定能很容易看懂 ArrayBlockingQueue 的源码，它采用一个 ReentrantLock 和相应的两个 Condition 来实现。

ArrayBlockingQueue 共有以下几个属性：

```
// 用于存放元素的数组
final Object[] items;
// 下一次读取操作的位置
int takeIndex;
// 下一次写入操作的位置
int putIndex;
// 队列中的元素数量
int count;

// 以下几个就是控制并发用的同步器
final ReentrantLock lock;
private final Condition notEmpty;
private final Condition notFull;

```

我们用个示意图来描述其同步机制：

![array-blocking-queue](https://javadoop.com/blogimages/java-concurrent-queue/array-blocking-queue.png)

ArrayBlockingQueue 实现并发同步的原理就是，读操作和写操作都需要获取到 AQS 独占锁才能进行操作。如果队列为空，这个时候读操作的线程进入到读线程队列排队，等待写线程写入新的元素，然后唤醒读线程队列的第一个等待线程。如果队列已满，这个时候写操作的线程进入到写线程队列排队，等待读线程将队列元素移除腾出空间，然后唤醒写线程队列的第一个等待线程。

对于 ArrayBlockingQueue，我们可以在构造的时候指定以下三个参数：

1.  队列容量，其限制了队列中最多允许的元素个数；
2.  指定独占锁是公平锁还是非公平锁。非公平锁的吞吐量比较高，公平锁可以保证每次都是等待最久的线程获取到锁；
3.  可以指定用一个集合来初始化，将此集合中的元素在构造方法期间就先添加到队列中。

更具体的源码我就不进行分析了，因为它就是 AbstractQueuedSynchronizer 中 Condition 的使用，感兴趣的读者请看我写的《[一行一行源码分析清楚 AbstractQueuedSynchronizer（二）](https://javadoop.com/post/AbstractQueuedSynchronizer-2/)》，因为只要看懂了那篇文章，ArrayBlockingQueue 的代码就没有分析的必要了，当然，如果你完全不懂 Condition，那么基本上也就可以说看不懂 ArrayBlockingQueue 的源码了。



## LinkedBlockingQueue

底层基于单向链表实现的阻塞队列，可以当做无界队列也可以当做有界队列来使用。看构造方法：

```
// 传说中的无界队列
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

```

```
// 传说中的有界队列
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}

```

我们看看这个类有哪些属性：

```
// 队列容量
private final int capacity;

// 队列中的元素数量
private final AtomicInteger count = new AtomicInteger(0);

// 队头
private transient Node<E> head;

// 队尾
private transient Node<E> last;

// take, poll, peek 等读操作的方法需要获取到这个锁
private final ReentrantLock takeLock = new ReentrantLock();

// 如果读操作的时候队列是空的，那么等待 notEmpty 条件
private final Condition notEmpty = takeLock.newCondition();

// put, offer 等写操作的方法需要获取到这个锁
private final ReentrantLock putLock = new ReentrantLock();

// 如果写操作的时候队列是满的，那么等待 notFull 条件
private final Condition notFull = putLock.newCondition();

```

这里用了两个锁，两个 Condition，简单介绍如下：

takeLock 和 notEmpty 怎么搭配：如果要获取（take）一个元素，需要获取 takeLock 锁，但是获取了锁还不够，如果队列此时为空，还需要队列不为空（notEmpty）这个条件（Condition）。

putLock 需要和 notFull 搭配：如果要插入（put）一个元素，需要获取 putLock 锁，但是获取了锁还不够，如果队列此时已满，还需要队列不是满的（notFull）这个条件（Condition）。

首先，这里用一个示意图来看看 LinkedBlockingQueue 的并发读写控制，然后再开始分析源码：

![linked-blocking-queue](https://javadoop.com/blogimages/java-concurrent-queue/linked-blocking-queue.png)

看懂这个示意图，源码也就简单了，读操作是排好队的，写操作也是排好队的，唯一的并发问题在于一个写操作和一个读操作同时进行，只要控制好这个就可以了。

先上构造方法：

```
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}

```

注意，这里会初始化一个空的头结点，那么第一个元素入队的时候，队列中就会有两个元素。读取元素时，也总是获取头节点后面的一个节点。count 的计数值不包括这个头节点。

我们来看下 put 方法是怎么将元素插入到队尾的：

```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // 如果你纠结这里为什么是 -1，可以看看 offer 方法。这就是个标识成功、失败的标志而已。
    int c = -1;
    Node<E> node = new Node(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 必须要获取到 putLock 才可以进行插入操作
    putLock.lockInterruptibly();
    try {
        // 如果队列满，等待 notFull 的条件满足。
        while (count.get() == capacity) {
            notFull.await();
        }
        // 入队
        enqueue(node);
        // count 原子加 1，c 还是加 1 前的值
        c = count.getAndIncrement();
        // 如果这个元素入队后，还有至少一个槽可以使用，调用 notFull.signal() 唤醒等待线程。
        // 哪些线程会等待在 notFull 这个 Condition 上呢？
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        // 入队后，释放掉 putLock
        putLock.unlock();
    }
    // 如果 c == 0，那么代表队列在这个元素入队前是空的（不包括head空节点），
    // 那么所有的读线程都在等待 notEmpty 这个条件，等待唤醒，这里做一次唤醒操作
    if (c == 0)
        signalNotEmpty();
}

// 入队的代码非常简单，就是将 last 属性指向这个新元素，并且让原队尾的 next 指向这个元素
// 这里入队没有并发问题，因为只有获取到 putLock 独占锁以后，才可以进行此操作
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}

// 元素入队后，如果需要，调用这个方法唤醒读线程来读
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}

```

我们再看看 take 方法：

```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    // 首先，需要获取到 takeLock 才能进行出队操作
    takeLock.lockInterruptibly();
    try {
        // 如果队列为空，等待 notEmpty 这个条件满足再继续执行
        while (count.get() == 0) {
            notEmpty.await();
        }
        // 出队
        x = dequeue();
        // count 进行原子减 1
        c = count.getAndDecrement();
        // 如果这次出队后，队列中至少还有一个元素，那么调用 notEmpty.signal() 唤醒其他的读线程
        if (c > 1)
            notEmpty.signal();
    } finally {
        // 出队后释放掉 takeLock
        takeLock.unlock();
    }
    // 如果 c == capacity，那么说明在这个 take 方法发生的时候，队列是满的
    // 既然出队了一个，那么意味着队列不满了，唤醒写线程去写
    if (c == capacity)
        signalNotFull();
    return x;
}
// 取队头，出队
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    // 之前说了，头结点是空的
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    // 设置这个为新的头结点
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
// 元素出队后，如果需要，调用这个方法唤醒写线程来写
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}

```

源码分析就到这里结束了吧，毕竟还是比较简单的源码，基本上只要读者认真点都看得懂。



## SynchronousQueue

它是一个特殊的队列，它的名字其实就蕴含了它的特征 - - 同步的队列。为什么说是同步的呢？这里说的并不是多线程的并发问题，而是因为当一个线程往队列中写入一个元素时，写入操作不会立即返回，需要等待另一个线程来将这个元素拿走；同理，当一个读线程做读操作的时候，同样需要一个相匹配的写线程的写操作。这里的 Synchronous 指的就是读线程和写线程需要同步，一个读线程匹配一个写线程。

我们比较少使用到 SynchronousQueue 这个类，不过它在线程池的实现类 ScheduledThreadPoolExecutor 中得到了应用，感兴趣的读者可以在看完这个后去看看相应的使用。

虽然上面我说了队列，但是 SynchronousQueue 的队列其实是虚的，其不提供任何空间（一个都没有）来存储元素。数据必须从某个写线程交给某个读线程，而不是写到某个队列中等待被消费。

你不能在 SynchronousQueue 中使用 peek 方法（在这里这个方法直接返回 null），peek 方法的语义是只读取不移除，显然，这个方法的语义是不符合 SynchronousQueue 的特征的。SynchronousQueue 也不能被迭代，因为根本就没有元素可以拿来迭代的。虽然 SynchronousQueue 间接地实现了 Collection 接口，但是如果你将其当做 Collection 来用的话，那么集合是空的。当然，这个类也是不允许传递 null 值的（并发包中的容器类好像都不支持插入 null 值，因为 null 值往往用作其他用途，比如用于方法的返回值代表操作失败）。

接下来，我们来看看具体的源码实现吧，它的源码不是很简单的那种，我们需要先搞清楚它的设计思想。

源码加注释大概有 1200 行，我们先看大框架：

```
// 构造时，我们可以指定公平模式还是非公平模式，区别之后再说
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue() : new TransferStack();
}
abstract static class Transferer {
    // 从方法名上大概就知道，这个方法用于转移元素，从生产者手上转到消费者手上
    // 也可以被动地，消费者调用这个方法来从生产者手上取元素
    // 第一个参数 e 如果不是 null，代表场景为：将元素从生产者转移给消费者
    // 如果是 null，代表消费者等待生产者提供元素，然后返回值就是相应的生产者提供的元素
    // 第二个参数代表是否设置超时，如果设置超时，超时时间是第三个参数的值
    // 返回值如果是 null，代表超时，或者中断。具体是哪个，可以通过检测中断状态得到。
    abstract Object transfer(Object e, boolean timed, long nanos);
}

```

Transferer 有两个内部实现类，是因为构造 SynchronousQueue 的时候，我们可以指定公平策略。公平模式意味着，所有的读写线程都遵守先来后到，FIFO 嘛，对应 TransferQueue。而非公平模式则对应 TransferStack。

SynchronousQueue采用队列TransferQueue来实现公平性策略，采用堆栈TransferStack来实现非公平性策略，他们两种都是通过链表实现的，其节点分别为QNode，SNode。TransferQueue和TransferStack在SynchronousQueue中扮演着非常重要的作用，SynchronousQueue的put、take操作都是委托这两个类来实现的。

![synchronous-queue](https://javadoop.com/blogimages/java-concurrent-queue/synchronous-queue.png)

我们先采用公平模式分析源码，然后再说说公平模式和非公平模式的区别。

接下来，我们看看 put 方法和 take 方法：

```
// 写入值
public void put(E o) throws InterruptedException {
    if (o == null) throw new NullPointerException();
    if (transferer.transfer(o, false, 0) == null) { // 1
        Thread.interrupted();
        throw new InterruptedException();
    }
}
// 读取值并移除
public E take() throws InterruptedException {
    Object e = transferer.transfer(null, false, 0); // 2
    if (e != null)
        return (E)e;
    Thread.interrupted();
    throw new InterruptedException();
}

```

我们看到，写操作 put(E o) 和读操作 take() 都是调用 Transferer.transfer(…) 方法，区别在于第一个参数是否为 null 值。

我们来看看 transfer 的设计思路，其基本算法如下：

1.  当调用这个方法时，如果队列是空的，或者队列中的节点和当前的线程操作类型一致（如当前操作是 put 操作，而队列中的元素也都是写线程）。这种情况下，将当前线程加入到等待队列即可。
2.  如果队列中有等待节点，而且与当前操作可以匹配（如队列中都是读操作线程，当前线程是写操作线程，反之亦然）。这种情况下，匹配等待队列的队头，出队，返回相应数据。

其实这里有个隐含的条件被满足了，队列如果不为空，肯定都是同种类型的节点，要么都是读操作，要么都是写操作。这个就要看到底是读线程积压了，还是写线程积压了。

我们可以假设出一个男女配对的场景：一个男的过来，如果一个人都没有，那么他需要等待；如果发现有一堆男的在等待，那么他需要排到队列后面；如果发现是一堆女的在排队，那么他直接牵走队头的那个女的。

既然这里说到了等待队列，我们先看看其实现，也就是 QNode:

```
static final class QNode {
    volatile QNode next;          // 可以看出来，等待队列是单向链表
    volatile Object item;         // CAS'ed to or from null
    volatile Thread waiter;       // 将线程对象保存在这里，用于挂起和唤醒
    final boolean isData;         // 用于判断是写线程节点(isData == true)，还是读线程节点

    QNode(Object item, boolean isData) {
        this.item = item;
        this.isData = isData;
    }
  ......

```

相信说了这么多以后，我们再来看 transfer 方法的代码就轻松多了。

```
/**
 * Puts or takes an item.
 */
Object transfer(Object e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin

        // 队列空，或队列中节点类型和当前节点一致，
        // 即我们说的第一种情况，将节点入队即可。读者要想着这块 if 里面方法其实就是入队
        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next;
            // t != tail 说明刚刚有节点入队，continue 即可
            if (t != tail)                  // inconsistent read
                continue;
            // 有其他节点入队，但是 tail 还是指向原来的，此时设置 tail 即可
            if (tn != null) {               // lagging tail
                // 这个方法就是：如果 tail 此时为 t 的话，设置为 tn
                advanceTail(t, tn);
                continue;
            }
            // 
            if (timed && nanos <= 0)        // can't wait
                return null;
            if (s == null)
                s = new QNode(e, isData);
            // 将当前节点，插入到 tail 的后面
            if (!t.casNext(null, s))        // failed to link in
                continue;

            // 将当前节点设置为新的 tail
            advanceTail(t, s);              // swing tail and wait
            // 看到这里，请读者先往下滑到这个方法，看完了以后再回来这里，思路也就不会断了
            Object x = awaitFulfill(s, e, timed, nanos);
            // 到这里，说明之前入队的线程被唤醒了，准备往下执行
            if (x == s) {                   // wait was cancelled
                clean(t, s);
                return null;
            }

            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? x : e;

        // 这里的 else 分支就是上面说的第二种情况，有相应的读或写相匹配的情况
        } else {                            // complementary-mode
            QNode m = h.next;               // node to fulfill
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;
            if (isData == (x != null) ||    // m already fulfilled
                x == m ||                   // m cancelled
                !m.casItem(x, e)) {         // lost CAS
                advanceHead(h, m);          // dequeue and retry
                continue;
            }

            advanceHead(h, m);              // successfully fulfilled
            LockSupport.unpark(m.waiter);
            return (x != null) ? x : e;
        }
    }
}

void advanceTail(QNode t, QNode nt) {
    if (tail == t)
        UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
}

```

```
// 自旋或阻塞，直到满足条件，这个方法返回
Object awaitFulfill(QNode s, Object e, boolean timed, long nanos) {

    long lastTime = timed ? System.nanoTime() : 0;
    Thread w = Thread.currentThread();
    // 判断需要自旋的次数，
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        // 如果被中断了，那么取消这个节点
        if (w.isInterrupted())
            // 就是将当前节点 s 中的 item 属性设置为 this
            s.tryCancel(e);
        Object x = s.item;
        // 这里是这个方法的唯一的出口
        if (x != e)
            return x;
        // 如果需要，检测是否超时
        if (timed) {
            long now = System.nanoTime();
            nanos -= now - lastTime;
            lastTime = now;
            if (nanos <= 0) {
                s.tryCancel(e);
                continue;
            }
        }
        if (spins > 0)
            --spins;
        // 如果自旋达到了最大的次数，那么检测
        else if (s.waiter == null)
            s.waiter = w;
        // 如果自旋到了最大的次数，那么线程挂起，等待唤醒
        else if (!timed)
            LockSupport.park(this);
        // spinForTimeoutThreshold 这个之前讲 AQS 的时候其实也说过，剩余时间小于这个阈值的时候，就
        // 不要进行挂起了，自旋的性能会比较好
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}

```

Doug Lea 的巧妙之处在于，将各个代码凑在了一起，使得代码非常简洁，当然也同时增加了我们的阅读负担，看代码的时候，还是得仔细想想各种可能的情况。

下面，再说说前面说的公平模式和非公平模式的区别。

相信大家心里面已经有了公平模式的工作流程的概念了，我就简单说说 TransferStack 的算法，就不分析源码了。

1.  当调用这个方法时，如果队列是空的，或者队列中的节点和当前的线程操作类型一致（如当前操作是 put 操作，而栈中的元素也都是写线程）。这种情况下，将当前线程加入到等待栈中，等待配对。然后返回相应的元素，或者如果被取消了的话，返回 null。
2.  如果栈中有等待节点，而且与当前操作可以匹配（如栈里面都是读操作线程，当前线程是写操作线程，反之亦然）。将当前节点压入栈顶，和栈中的节点进行匹配，然后将这两个节点出栈。配对和出栈的动作其实也不是必须的，因为下面的一条会执行同样的事情。
3.  如果栈顶是进行匹配而入栈的节点，帮助其进行匹配并出栈，然后再继续操作。

应该说，TransferStack 的源码要比 TransferQueue 的复杂一些，如果读者感兴趣，请自行进行源码阅读。



## PriorityBlockingQueue

带排序的 BlockingQueue 实现，其并发控制采用的是 ReentrantLock，队列为无界队列（ArrayBlockingQueue 是有界队列，LinkedBlockingQueue 也可以通过在构造函数中传入 capacity 指定队列最大的容量，但是 PriorityBlockingQueue 只能指定初始的队列大小，后面插入元素的时候，如果空间不够的话会自动扩容）。

简单地说，它就是 PriorityQueue 的线程安全版本。不可以插入 null 值，同时，插入队列的对象必须是可比较大小的（comparable），否则报 ClassCastException 异常。它的插入操作 put 方法不会 block，因为它是无界队列（take 方法在队列为空的时候会阻塞）。

它的源码相对比较简单，本节将介绍其核心源码部分。

我们来看看它有哪些属性：

```
// 构造方法中，如果不指定大小的话，默认大小为 11
private static final int DEFAULT_INITIAL_CAPACITY = 11;
// 数组的最大容量
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

// 这个就是存放数据的数组
private transient Object[] queue;

// 队列当前大小
private transient int size;

// 大小比较器，如果按照自然序排序，那么此属性可设置为 null
private transient Comparator<? super E> comparator;

// 并发控制所用的锁，所有的 public 且涉及到线程安全的方法，都必须先获取到这个锁
private final ReentrantLock lock;

// 这个很好理解，其实例由上面的 lock 属性创建
private final Condition notEmpty;

// 这个也是用于锁，用于数组扩容的时候，需要先获取到这个锁，才能进行扩容操作
// 其使用 CAS 操作
private transient volatile int allocationSpinLock;

// 用于序列化和反序列化的时候用，对于 PriorityBlockingQueue 我们应该比较少使用到序列化
private PriorityQueue q;

```

此类实现了 Collection 和 Iterator 接口中的所有接口方法，对其对象进行迭代并遍历时，不能保证有序性。如果你想要实现有序遍历，建议采用 Arrays.sort(queue.toArray()) 进行处理。PriorityBlockingQueue 提供了 drainTo 方法用于将部分或全部元素有序地填充（准确说是转移，会删除原队列中的元素）到另一个集合中。还有一个需要说明的是，如果两个对象的优先级相同（compare 方法返回 0），此队列并不保证它们之间的顺序。

PriorityBlockingQueue 使用了基于数组的二叉堆来存放元素，所有的 public 方法采用同一个 lock 进行并发控制。

二叉堆：一颗完全二叉树，它非常适合用数组进行存储，对于数组中的元素 a[i]，其左子节点为 a[2_i+1]，其右子节点为 a[2\_i + 2]，其父节点为 a[(i-1)/2]，其堆序性质为，每个节点的值都小于其左右子节点的值。二叉堆中最小的值就是根节点，但是删除根节点是比较麻烦的，因为需要调整树。

简单用个图解释一下二叉堆，我就不说太多专业的严谨的术语了，这种数据结构的优点是一目了然的，最小的元素一定是根元素，它是一棵满的树，除了最后一层，最后一层的节点从左到右紧密排列。

![priority-blocking-queue-1](https://javadoop.com/blogimages/java-concurrent-queue/priority-blocking-queue-1.png)

下面开始 PriorityBlockingQueue 的源码分析，首先我们来看看构造方法:

```
// 默认构造方法，采用默认值(11)来进行初始化
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
// 指定数组的初始大小
public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}
// 指定比较器
public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}
// 在构造方法中就先填充指定的集合中的元素
public PriorityBlockingQueue(Collection<? extends E> c) {
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    // 
    boolean heapify = true; // true if not known to be in heap order
    boolean screen = true;  // true if must screen for nulls
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        heapify = false;
    }
    else if (c instanceof PriorityBlockingQueue<?>) {
        PriorityBlockingQueue<? extends E> pq =
            (PriorityBlockingQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        screen = false;
        if (pq.getClass() == PriorityBlockingQueue.class) // exact match
            heapify = false;
    }
    Object[] a = c.toArray();
    int n = a.length;
    // If c.toArray incorrectly doesn't return Object[], copy it.
    if (a.getClass() != Object[].class)
        a = Arrays.copyOf(a, n, Object[].class);
    if (screen && (n == 1 || this.comparator != null)) {
        for (int i = 0; i < n; ++i)
            if (a[i] == null)
                throw new NullPointerException();
    }
    this.queue = a;
    this.size = n;
    if (heapify)
        heapify();
}

```

接下来，我们来看看其内部的自动扩容实现：

```
private void tryGrow(Object[] array, int oldCap) {
    // 这边做了释放锁的操作
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    // 用 CAS 操作将 allocationSpinLock 由 0 变为 1，也算是获取锁
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            // 如果节点个数小于 64，那么增加的 oldCap + 2 的容量
            // 如果节点数大于等于 64，那么增加 oldCap 的一半
            // 所以节点数较小时，增长得快一些
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) :
                                   (oldCap >> 1));
            // 这里有可能溢出
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            // 如果 queue != array，那么说明有其他线程给 queue 分配了其他的空间
            if (newCap > oldCap && queue == array)
                // 分配一个新的大数组
                newArray = new Object[newCap];
        } finally {
            // 重置，也就是释放锁
            allocationSpinLock = 0;
        }
    }
    // 如果有其他的线程也在做扩容的操作
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    // 重新获取锁
    lock.lock();
    // 将原来数组中的元素复制到新分配的大数组中
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}

```

扩容方法对并发的控制也非常的巧妙，释放了原来的独占锁 lock，这样的话，扩容操作和读操作可以同时进行，提高吞吐量。

下面，我们来分析下写操作 put 方法和读操作 take 方法。

```
public void put(E e) {
    // 直接调用 offer 方法，因为前面我们也说了，在这里，put 方法不会阻塞
    offer(e); 
}
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    // 首先获取到独占锁
    lock.lock();
    int n, cap;
    Object[] array;
    // 如果当前队列中的元素个数 >= 数组的大小，那么需要扩容了
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        // 节点添加到二叉堆中
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        // 更新 size
        size = n + 1;
        // 唤醒等待的读线程
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}

```

对于二叉堆而言，插入一个节点是简单的，插入的节点如果比父节点小，交换它们，然后继续和父节点比较。

```
// 这个方法就是将数据 x 插入到数组 array 的位置 k 处，然后再调整树
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        // 二叉堆中 a[k] 节点的父节点位置
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = key;
}

```

我们用图来示意一下，我们接下来要将 11 插入到队列中，看看 siftUp 是怎么操作的。

![priority-blocking-queue-2](https://javadoop.com/blogimages/java-concurrent-queue/priority-blocking-queue-2.png)

我们再看看 take 方法：

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 独占锁
    lock.lockInterruptibly();
    E result;
    try {
        // dequeue 出队
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}

```

```
private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        // 队头，用于返回
        E result = (E) array[0];
        // 队尾元素先取出
        E x = (E) array[n];
        // 队尾置空
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}

```

dequeue 方法返回队头，并调整二叉堆的树，调用这个方法必须先获取独占锁。

废话不多说，出队是非常简单的，因为队头就是最小的元素，对应的是数组的第一个元素。难点是队头出队后，需要调整树。

```
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        // 这里得到的 half 肯定是非叶节点
        // a[n] 是最后一个元素，其父节点是 a[(n-1)/2]。所以 n >>> 1 代表的节点肯定不是叶子节点
        // 下面，我们结合图来一行行分析，这样比较直观简单
        // 此时 k 为 0, x 为 17，n 为 9
        int half = n >>> 1; // 得到 half = 4
        while (k < half) {
            // 先取左子节点
            int child = (k << 1) + 1; // 得到 child = 1
            Object c = array[child];  // c = 12
            int right = child + 1;  // right = 2
            // 如果右子节点存在，而且比左子节点小
            // 此时 array[right] = 20，所以条件不满足
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            // key = 17, c = 12，所以条件不满足
            if (key.compareTo((T) c) <= 0)
                break;
            // 把 12 填充到根节点
            array[k] = c;
            // k 赋值后为 1
            k = child;
            // 一轮过后，我们发现，12 左边的子树和刚刚的差不多，都是缺少根节点，接下来处理就简单了
        }
        array[k] = key;
    }
}

```

![priority-blocking-queue-3](https://javadoop.com/blogimages/java-concurrent-queue/priority-blocking-queue-3.png)

记住二叉堆是一棵完全二叉树，那么根节点 10 拿掉后，最后面的元素 17 必须找到合适的地方放置。首先，17 和 10 不能直接交换，那么先将根节点 10 的左右子节点中较小的节点往上滑，即 12 往上滑，然后原来 12 留下了一个空节点，然后再把这个空节点的较小的子节点往上滑，即 13 往上滑，最后，留出了位子，17 补上即可。

我稍微调整下这个树，以便读者能更明白：

![priority-blocking-queue-4](https://javadoop.com/blogimages/java-concurrent-queue/priority-blocking-queue-4.png)

好了， PriorityBlockingQueue 我们也说完了。



## DelayQueue

> 原文出处[http://cmsblogs.com/](http://cmsblogs.com/) 『chenssy』

DelayQueue是一个支持延时获取元素的无界阻塞队列。里面的元素全部都是“可延期”的元素，列头的元素是最先“到期”的元素，如果队列里面没有元素到期，是不能从列头获取元素的，哪怕有元素也不行。也就是说只有在延迟期到时才能够从队列中取元素。

DelayQueue主要用于两个方面：
- 缓存：清掉缓存中超时的缓存数据
- 任务超时处理

DelayQueue

DelayQueue实现的关键主要有如下几个：

1.  可重入锁ReentrantLock
2.  用于阻塞和通知的Condition对象
3.  根据Delay时间排序的优先级队列：PriorityQueue
4.  用于优化阻塞通知的线程元素leader

ReentrantLock、Condition这两个对象就不需要阐述了，他是实现整个BlockingQueue的核心。PriorityQueue是一个支持优先级线程排序的队列（参考[【死磕Java并发】-----J.U.C之阻塞队列：PriorityBlockingQueue](http://cmsblogs.com/?p=2407)），leader后面阐述。这里我们先来了解Delay，他是实现延时操作的关键。

Delayed

Delayed接口是用来标记那些应该在给定延迟时间之后执行的对象，它定义了一个long getDelay(TimeUnit unit)方法，该方法返回与此对象相关的的剩余时间。同时实现该接口的对象必须定义一个compareTo 方法，该方法提供与此接口的 getDelay 方法一致的排序。

```
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}

```

如何使用该接口呢？上面说的非常清楚了，实现该接口的getDelay()方法，同时定义compareTo()方法即可。

内部结构

先看DelayQueue的定义：

```
    public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
            implements BlockingQueue<E> {
        /** 可重入锁 */
        private final transient ReentrantLock lock = new ReentrantLock();
        /** 支持优先级的BlockingQueue */
        private final PriorityQueue<E> q = new PriorityQueue<E>();
        /** 用于优化阻塞 */
        private Thread leader = null;
        /** Condition */
        private final Condition available = lock.newCondition();

        /**
         * 省略很多代码
         */
    }

```

看了DelayQueue的内部结构就对上面几个关键点一目了然了，但是这里有一点需要注意，DelayQueue的元素都必须继承Delayed接口。同时也可以从这里初步理清楚DelayQueue内部实现的机制了：以支持优先级无界队列的PriorityQueue作为一个容器，容器里面的元素都应该实现Delayed接口，在每次往优先级队列中添加元素时以元素的过期时间作为排序条件，最先过期的元素放在优先级最高。

offer()

```
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 向 PriorityQueue中插入元素
            q.offer(e);
            // 如果当前元素的对首元素（优先级最高），leader设置为空，唤醒所有等待线程
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            // 无界队列，永远返回true
            return true;
        } finally {
            lock.unlock();
        }
    }

```

offer(E e)就是往PriorityQueue中添加元素，具体可以参考（[【死磕Java并发】-----J.U.C之阻塞队列：PriorityBlockingQueue](http://cmsblogs.com/?p=2407)）。整个过程还是比较简单，但是在判断当前元素是否为对首元素，如果是的话则设置leader=null，这是非常关键的一个步骤，后面阐述。

take()

```
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                // 对首元素
                E first = q.peek();
                // 对首为空，阻塞，等待off()操作唤醒
                if (first == null)
                    available.await();
                else {
                    // 获取对首元素的超时时间
                    long delay = first.getDelay(NANOSECONDS);
                    // <=0 表示已过期，出对，return
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    // leader != null 证明有其他线程在操作，阻塞
                    if (leader != null)
                        available.await();
                    else {
                        // 否则将leader 设置为当前线程，独占
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            // 超时阻塞
                            available.awaitNanos(delay);
                        } finally {
                            // 释放leader
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            // 唤醒阻塞线程
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }

```

首先是获取对首元素，如果对首元素的延时时间 delay <= 0 ，则可以出对了，直接return即可。否则设置first = null，这里设置为null的主要目的是为了避免内存泄漏。如果 leader != null 则表示当前有线程占用，则阻塞，否则设置leader为当前线程，然后调用awaitNanos()方法超时等待。

first = null

这里为什么如果不设置first = null，则会引起内存泄漏呢？线程A到达，列首元素没有到期，设置leader = 线程A，这是线程B来了因为leader != null，则会阻塞，线程C一样。假如线程阻塞完毕了，获取列首元素成功，出列。这个时候列首元素应该会被回收掉，但是问题是它还被线程B、线程C持有着，所以不会回收，这里只有两个线程，如果有线程D、线程E...呢？这样会无限期的不能回收，就会造成内存泄漏。

这个入队、出对过程和其他的阻塞队列没有很大区别，无非是在出对的时候增加了一个到期时间的判断。同时通过leader来减少不必要阻塞。

ConcurrentLinkedQueue



> 原文出处[http://cmsblogs.com/](http://cmsblogs.com/) 『chenssy』

要实现一个线程安全的队列有两种方式：阻塞和非阻塞。阻塞队列无非就是锁的应用，而非阻塞则是CAS算法的应用。下面我们就开始一个非阻塞算法的研究：CoucurrentLinkedQueue。

ConcurrentLinkedQueue是一个基于链接节点的无边界的线程安全队列，它采用FIFO原则对元素进行排序。采用“wait-free”算法（即CAS算法）来实现的。

CoucurrentLinkedQueue规定了如下几个不变性：

1.  在入队的最后一个元素的next为null
2.  队列中所有未删除的节点的item都不能为null且都能从head节点遍历到
3.  对于要删除的节点，不是直接将其设置为null，而是先将其item域设置为null（迭代器会跳过item为null的节点）
4.  允许head和tail更新滞后。这是什么意思呢？意思就说是head、tail不总是指向第一个元素和最后一个元素（后面阐述）。

head的不变性和可变性：

*   不变性
    1.  所有未删除的节点都可以通过head节点遍历到
    2.  head不能为null
    3.  head节点的next不能指向自身
*   可变性
    1.  head的item可能为null，也可能不为null
        2.允许tail滞后head，也就是说调用succc()方法，从head不可达tail

tail的不变性和可变性

*   不变性
    1.  tail不能为null
*   可变性
    1.  tail的item可能为null，也可能不为null
    2.  tail节点的next域可以指向自身
        3.允许tail滞后head，也就是说调用succc()方法，从head不可达tail

这些特性是否已经晕了？没关系，我们看下面的源码分析就可以理解这些特性了。

## ConcurrentLinkedQueue源码分析

CoucurrentLinkedQueue的结构由head节点和tail节点组成，每个节点由节点元素item和指向下一个节点的next引用组成，而节点与节点之间的关系就是通过该next关联起来的，从而组成一张链表的队列。节点Node为ConcurrentLinkedQueue的内部类，定义如下：

<pre name="code">  private  static  class  Node<E>  {  /** 节点元素域 */  volatile E item;  volatile  Node<E>  next;  //初始化,获得item 和 next 的偏移量,为后期的CAS做准备  Node(E item)  { UNSAFE.putObject(this, itemOffset, item);  }  boolean casItem(E cmp, E val)  {  return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);  }  void lazySetNext(Node<E> val)  { UNSAFE.putOrderedObject(this, nextOffset, val);  }  boolean casNext(Node<E> cmp,  Node<E> val)  {  return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);  }  // Unsafe mechanics  private  static  final sun.misc.Unsafe UNSAFE;  /** 偏移量 */  private  static  final  long itemOffset;  /** 下一个元素的偏移量 */  private  static  final  long nextOffset;  static  {  try  { UNSAFE = sun.misc.Unsafe.getUnsafe();  Class<?> k =  Node.class; itemOffset = UNSAFE.objectFieldOffset (k.getDeclaredField("item")); nextOffset = UNSAFE.objectFieldOffset (k.getDeclaredField("next"));  }  catch  (Exception e)  {  throw  new  Error(e);  }  }  }</pre>

### 入列

入列，我们认为是一个非常简单的过程：tail节点的next执行新节点，然后更新tail为新节点即可。从单线程角度我们这么理解应该是没有问题的，但是多线程呢？如果一个线程正在进行插入动作，那么它必须先获取尾节点，然后设置尾节点的下一个节点为当前节点，但是如果已经有一个线程刚刚好完成了插入，那么尾节点是不是发生了变化？对于这种情况ConcurrentLinkedQueue怎么处理呢？我们先看源码：

offer(E e)：将指定元素插入都队列尾部：

<pre name="code">public  boolean offer(E e)  {  //检查节点是否为null checkNotNull(e);  // 创建新节点  final  Node<E> newNode =  new  Node<E>(e);  //死循环 直到成功为止  for  (Node<E> t = tail, p = t;;)  {  Node<E> q = p.next;  // q == null 表示 p已经是最后一个节点了，尝试加入到队列尾  // 如果插入失败，则表示其他线程已经修改了p的指向  if  (q ==  null)  {  // --- 1  // casNext：t节点的next指向当前节点  // casTail：设置tail 尾节点  if  (p.casNext(null, newNode))  {  // --- 2  // node 加入节点后会导致tail距离最后一个节点相差大于一个，需要更新tail  if  (p != t)  // --- 3 casTail(t, newNode);  // --- 4  return  true;  }  }  // p == q 等于自身  else  if  (p == q)  // --- 5  // p == q 代表着该节点已经被删除了  // 由于多线程的原因，我们offer()的时候也会poll()，如果offer()的时候正好该节点已经poll()了  // 那么在poll()方法中的updateHead()方法会将head指向当前的q，而把p.next指向自己，即：p.next == p  // 这样就会导致tail节点滞后head（tail位于head的前面），则需要重新设置p p =  (t !=  (t = tail))  ? t : head;  // --- 6  // tail并没有指向尾节点  else  // tail已经不是最后一个节点，将p指向最后一个节点 p =  (p != t && t !=  (t = tail))  ? t : q;  // --- 7  }  }</pre>

光看源码还是有点儿迷糊的，插入节点一次分析就会明朗很多。

初始化

ConcurrentLinkedQueue初始化时head、tail存储的元素都为null，且head等于tail：

[![201703160001](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160001_thumb.png "201703160001")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160001.png)

添加元素A

按照程序分析：第一次插入元素A，head = tail = dummyNode，所有q = p.next = null，直接走步骤2：p.casNext(null, newNode)，由于 p == t成立，所以不会执行步骤3：casTail(t, newNode)，直接return。插入A节点后如下：

[![201703160002](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160002_thumb.png "201703160002")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160002.png)

添加元素B

q = p.next = A ,p = tail = dummyNode，所以直接跳到步骤7：p = (p != t && t != (t = tail)) ? t : q;。此时p = q，然后进行第二次循环 q = p.next = null，步骤2：p == null成立，将该节点插入，因为p = q，t = tail，所以步骤3：p != t 成立，执行步骤4：casTail(t, newNode)，然后return。如下：
[![201703160003](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160003_thumb.png "201703160003")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160003.png)

添加节点C

此时t = tail ,p = t，q = p.next = null，和插入元素A无异，如下：
[![201703160004](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160004_thumb.png "201703160004")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160004.png)

这里整个offer()过程已经分析完成了，可能p == q 有点儿难理解，p 不是等于q.next么，怎么会有p == q呢？这个疑问我们在出列poll()中分析

### 出列

ConcurrentLinkedQueue提供了poll()方法进行出列操作。入列主要是涉及到tail，出列则涉及到head。我们先看源码：

<pre name="code">public E poll()  {  // 如果出现p被删除的情况需要从head重新开始 restartFromHead:  // 这是什么语法？真心没有见过  for  (;;)  {  for  (Node<E> h = head, p = h, q;;)  {  // 节点 item E item = p.item;  // item 不为null，则将item 设置为null  if  (item !=  null  && p.casItem(item,  null))  {  // --- 1  // p != head 则更新head  if  (p != h)  // --- 2  // p.next != null，则将head更新为p.next ,否则更新为p updateHead(h,  ((q = p.next)  !=  null)  ? q : p);  // --- 3  return item;  }  // p.next == null 队列为空  else  if  ((q = p.next)  ==  null)  {  // --- 4 updateHead(h, p);  return  null;  }  // 当一个线程在poll的时候，另一个线程已经把当前的p从队列中删除——将p.next = p，p已经被移除不能继续，需要重新开始  else  if  (p == q)  // --- 5  continue restartFromHead;  else p = q;  // --- 6  }  }  }</pre>

这个相对于offer()方法而言会简单些，里面有一个很重要的方法：updateHead()，该方法用于CAS更新head节点，如下：

<pre name="code">  final  void updateHead(Node<E> h,  Node<E> p)  {  if  (h != p && casHead(h, p)) h.lazySetNext(h);  }</pre>

我们先将上面offer()的链表poll()掉，添加A、B、C节点结构如下：

[![201703160004_2](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160004_2_thumb.png "201703160004_2")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160004_2.png)

poll A

head = dumy，p = head， item = p.item = null，步骤1不成立，步骤4：(q = p.next) == null不成立，p.next = A，跳到步骤6，下一个循环，此时p = A，所以步骤1 item != null，进行p.casItem(item, null)成功，此时p == A != h，所以执行步骤3：updateHead(h, ((q = p.next) != null) ? q : p)，q = p.next = B != null，则将head CAS更新成B，如下：

[![201703160005](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160005_thumb.png "201703160005")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160005.png)

poll B

head = B ， p = head = B，item = p.item = B，步骤成立，步骤2：p != h 不成立，直接return，如下：

[![201703160006](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160006_thumb.png "201703160006")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160006.png)

poll C

head = dumy ，p = head = dumy，tiem = p.item = null，步骤1不成立，跳到步骤4：(q = p.next) == null，不成立，然后跳到步骤6，此时，p = q = C，item = C(item)，步骤1成立，所以讲C（item）设置为null，步骤2：p != h成立，执行步骤3：updateHead(h, ((q = p.next) != null) ? q : p)，如下：

[![201703160007](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160007_thumb.png "201703160007")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160007.png)

看到这里是不是一目了然了，在这里我们再来分析offer()的步骤5：

<pre name="code">else  if(p == q){ p =  (t !=  (t = tail))? t : head;  }</pre>

ConcurrentLinkedQueue中规定，p == q表明，该节点已经被删除了，也就说tail滞后于head，head无法通过succ()方法遍历到tail，怎么做？ (t != (t = tail))? t : head;（这段代码的可读性实在是太差了，真他妈难理解：不知道是否可以理解为t != tail ? tail : head）这段代码主要是来判读tail节点是否已经发生了改变，如果发生了改变，则说明tail已经重新定位了，只需要重新找到tail即可，否则就只能指向head了。

就上面那个我们再次插入一个元素D。则p = head，q = p.next = null，执行步骤1： q = null且 p != t ，所以执行步骤4:，如下：

[![201703160008_2](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160008_2_thumb.png "201703160008_2")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160008_2.png)

再插入元素E，q = p.next = null，p == t，所以插入E后如下：

[![201703160009_2](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160009_2_thumb.png "201703160009_2")](http://cmsblogs.qiniudn.com/wp-content/uploads/2017/07/201703160009_2.png)

到这里ConcurrentLinkedQueue的整个入列、出列都已经分析完毕了，对于ConcurrentLinkedQueue LZ真心感觉难看懂，看懂之后也感叹设计得太精妙了，利用CAS来完成数据操作，同时允许队列的不一致性，这种弱一致性确实是非常强大。再次感叹Doug Lea的天才。

### LinkedTransferQueue



> 原文出处[http://cmsblogs.com/](http://cmsblogs.com/) 『chenssy』

前面提到的各种BlockingQueue对读或者写都是锁上整个队列，在并发量大的时候，各种锁是比较耗资源和耗时间的，而前面的SynchronousQueue虽然不会锁住整个队列，但它是一个没有容量的“队列”，那么有没有这样一种队列，它即可以像其他的BlockingQueue一样有容量又可以像SynchronousQueue一样不会锁住整个队列呢？有！答案就是LinkedTransferQueue。

LinkedTransferQueue是基于链表的FIFO无界阻塞队列，它出现在JDK7中。Doug Lea 大神说LinkedTransferQueue是一个聪明的队列。它是[ConcurrentLinkedQueue](http://cmsblogs.com/?p=2353)、[SynchronousQueue](http://cmsblogs.com/?p=2418) (公平模式下)、无界的[LinkedBlockingQueues](http://cmsblogs.com/?p=2381)等的超集。既然这么牛逼，那势必要弄清楚其中的原理了。

## LinkedTransferQueue

看源码之前我们先稍微了解下它的原理，这样看源码就会有迹可循了。

LinkedTransferQueue采用一种预占模式。什么意思呢？有就直接拿走，没有就占着这个位置直到拿到或者超时或者中断。即消费者线程到队列中取元素时，如果发现队列为空，则会生成一个null节点，然后park住等待生产者。后面如果生产者线程入队时发现有一个null元素节点，这时生产者就不会入列了，直接将元素填充到该节点上，唤醒该节点的线程，被唤醒的消费者线程拿东西走人。是不是有点儿[SynchronousQueue](http://cmsblogs.com/?p=2418)的味道？

### 结构

LinkedTransferQueue与其他的BlockingQueue一样，同样继承AbstractQueue类，但是它实现了TransferQueue，TransferQueue接口继承BlockingQueue，所以TransferQueue算是对BlockingQueue一种扩充，该接口提供了一整套的transfer接口：

```
    public interface TransferQueue<E> extends BlockingQueue<E> {

        /**
         * 若当前存在一个正在等待获取的消费者线程（使用take()或者poll()函数），使用该方法会即刻转移/传输对象元素e；
         * 若不存在，则返回false，并且不进入队列。这是一个不阻塞的操作
         */
        boolean tryTransfer(E e);

        /**
         * 若当前存在一个正在等待获取的消费者线程，即立刻移交之；
         * 否则，会插入当前元素e到队列尾部，并且等待进入阻塞状态，到有消费者线程取走该元素
         */
        void transfer(E e) throws InterruptedException;

        /**
         * 若当前存在一个正在等待获取的消费者线程，会立即传输给它;否则将插入元素e到队列尾部，并且等待被消费者线程获取消费掉；
         * 若在指定的时间内元素e无法被消费者线程获取，则返回false，同时该元素被移除。
         */
        boolean tryTransfer(E e, long timeout, TimeUnit unit)
                throws InterruptedException;

        /**
         * 判断是否存在消费者线程
         */
        boolean hasWaitingConsumer();

        /**
         * 获取所有等待获取元素的消费线程数量
         */
        int getWaitingConsumerCount();
    }

```

相对于其他的BlockingQueue，LinkedTransferQueue就多了上面几个方法。这几个方法在LinkedTransferQueue中起到了核心作用。

LinkedTransferQueue定义的变量如下：

```
    // 判断是否为多核
    private static final boolean MP =
            Runtime.getRuntime().availableProcessors() > 1;

    // 自旋次数
    private static final int FRONT_SPINS   = 1 << 7;

    // 前驱节点正在处理，当前节点需要自旋的次数
    private static final int CHAINED_SPINS = FRONT_SPINS >>> 1;

    static final int SWEEP_THRESHOLD = 32;

    // 头节点
    transient volatile Node head;

    // 尾节点
    private transient volatile Node tail;

    // 删除节点失败的次数
    private transient volatile int sweepVotes;

    /*
     * 调用xfer()方法时需要传入,区分不同处理
     * xfer()方法是LinkedTransferQueue的最核心的方法
     */
    private static final int NOW   = 0; // for untimed poll, tryTransfer
    private static final int ASYNC = 1; // for offer, put, add
    private static final int SYNC  = 2; // for transfer, take
    private static final int TIMED = 3; // for timed poll, tryTransfer

```

### Node节点

Node节点由四个部分构成：

*   isData：表示该节点是存放数据还是获取数据
*   item：存放数据，isData为false时，该节点为null，为true时，匹配后，该节点会置为null
*   next：指向下一个节点
*   waiter：park住消费者线程，线程就放在这里

结构如下：

![这里写图片描述](https://img-blog.csdn.net/20170924210642971?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
源码如下：

```
    static final class Node {
        // 表示该节点是存放数据还是获取数据
        final boolean isData;
        // 存放数据，isData为false时，该节点为null，为true时，匹配后，该节点会置为null
        volatile Object item;
        //指向下一个节点
        volatile Node next;

        // park住消费者线程，线程就放在这里
        volatile Thread waiter; // null until waiting

        /**
         * CAS Next域
         */
        final boolean casNext(Node cmp, Node val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        /**
         * CAS itme域
         */
        final boolean casItem(Object cmp, Object val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        /**
         * 构造函数
         */
        Node(Object item, boolean isData) {
            UNSAFE.putObject(this, itemOffset, item); // relaxed write
            this.isData = isData;
        }

        /**
         * 将next域指向自身，其实就是剔除节点
         */
        final void forgetNext() {
            UNSAFE.putObject(this, nextOffset, this);
        }

        /**
         *  匹配过或节点被取消的时候会调用
         */
        final void forgetContents() {
            UNSAFE.putObject(this, itemOffset, this);
            UNSAFE.putObject(this, waiterOffset, null);
        }

        /**
         * 校验节点是否匹配过，如果匹配做取消了，item则会发生变化
         */
        final boolean isMatched() {
            Object x = item;
            return (x == this) || ((x == null) == isData);
        }

        /**
         * 是否是一个未匹配的请求节点
         * 如果是的话isData应为false，item == null，因位如果匹配了，item则会有值
         */
        final boolean isUnmatchedRequest() {
            return !isData && item == null;
        }

        /**
         * 如给定节点类型不能挂在当前节点后返回true
         */
        final boolean cannotPrecede(boolean haveData) {
            boolean d = isData;
            Object x;
            return d != haveData && (x = item) != this && (x != null) == d;
        }

        /**
         * 匹配一个数据节点
         */
        final boolean tryMatchData() {
            // assert isData;
            Object x = item;
            if (x != null && x != this && casItem(x, null)) {
                LockSupport.unpark(waiter);
                return true;
            }
            return false;
        }

        private static final long serialVersionUID = -3375979862319811754L;

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;
        private static final long waiterOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("next"));
                waiterOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("waiter"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

```

节点Node为LinkedTransferQueue的内部类，其内部结构和公平方式的SynchronousQueue差不多，里面也同样提供了一些很重要的方法。

### put操作

LinkedTransferQueue提供了add、put、offer三类方法，用于将元素插入队列中，如下：

```
    public void put(E e) {
        xfer(e, true, ASYNC, 0);
    }

    public boolean offer(E e, long timeout, TimeUnit unit) {
        xfer(e, true, ASYNC, 0);
        return true;
    }

    public boolean offer(E e) {
        xfer(e, true, ASYNC, 0);
        return true;
    }

    public boolean add(E e) {
        xfer(e, true, ASYNC, 0);
        return true;
    }

```

由于LinkedTransferQueue是无界的，不会阻塞，所以在调用xfer方法是传入的是ASYNC，同时直接返回true.

### take操作

LinkedTransferQueue提供了poll、take方法用于出列元素：

```
    public E take() throws InterruptedException {
        E e = xfer(null, false, SYNC, 0);
        if (e != null)
            return e;
        Thread.interrupted();
        throw new InterruptedException();
    }

    public E poll() {
        return xfer(null, false, NOW, 0);
    }

    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E e = xfer(null, false, TIMED, unit.toNanos(timeout));
        if (e != null || !Thread.interrupted())
            return e;
        throw new InterruptedException();
    }

```

这里和put操作有点不一样，take()方法传入的是SYNC，阻塞。poll()传入的是NOW，poll(long timeout, TimeUnit unit)则是传入TIMED。

### tranfer操作

实现TransferQueue接口，就要实现它的方法：

```
public boolean tryTransfer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (xfer(e, true, TIMED, unit.toNanos(timeout)) == null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}

public void transfer(E e) throws InterruptedException {
    if (xfer(e, true, SYNC, 0) != null) {
        Thread.interrupted(); // failure possible only due to interrupt
        throw new InterruptedException();
    }
}

public boolean tryTransfer(E e) {
    return xfer(e, true, NOW, 0) == null;
}

```

### xfer()

通过上面几个核心方法的源码我们清楚可以看到，最终都是调用xfer()方法，该方法接受四个参数，item或者null的E，put操作为true、take操作为false的havaData，how（有四个值NOW, ASYNC, SYNC, or TIMED，分别表示不同的操作），超时nanos。

```
    private E xfer(E e, boolean haveData, int how, long nanos) {

        // havaData为true，但是e == null 抛出空指针
        if (haveData && (e == null))
            throw new NullPointerException();
        Node s = null;                        // the node to append, if needed

        retry:
        for (;;) {

            // 从首节点开始匹配
            // p == null 队列为空
            for (Node h = head, p = h; p != null;) {

                // 模型，request or data
                boolean isData = p.isData;
                // item域
                Object item = p.item;

                // 找到一个没有匹配的节点
                // item != p 也就是自身，则表示没有匹配过
                // (item != null) == isData，表示模型符合
                if (item != p && (item != null) == isData) {

                    // 节点类型和待处理类型一致，这样肯定是不能匹配的
                    if (isData == haveData)   // can't match
                        break;
                    // 匹配，将E加入到item域中
                    // 如果p 的item为data，那么e为null,如果p的item为null，那么e为data
                    if (p.casItem(item, e)) { // match
                        //
                        for (Node q = p; q != h;) {
                            Node n = q.next;  // update by 2 unless singleton
                            if (head == h && casHead(h, n == null ? q : n)) {
                                h.forgetNext();
                                break;
                            }                 // advance and retry
                            if ((h = head)   == null ||
                                    (q = h.next) == null || !q.isMatched())
                                break;        // unless slack < 2
                        }

                        // 匹配后唤醒p的waiter线程;reservation则叫人收货，data则叫null收货
                        LockSupport.unpark(p.waiter);
                        return LinkedTransferQueue.<E>cast(item);
                    }
                }
                // 如果已经匹配了则向前推进
                Node n = p.next;
                // 如果p的next指向p本身，说明p节点已经有其他线程处理过了，只能从head重新开始
                p = (p != n) ? n : (h = head); // Use head if p offlist
            }

            // 如果没有找到匹配的节点，则进行处理
            // NOW为untimed poll, tryTransfer，不需要入队
            if (how != NOW) {                 // No matches available
                // s == null，新建一个节点
                if (s == null)
                    s = new Node(e, haveData);
                // 入队，返回前驱节点
                Node pred = tryAppend(s, haveData);
                // 返回的前驱节点为null，那就是有race，被其他的抢了，那就continue 整个for
                if (pred == null)
                    continue retry;

                // ASYNC不需要阻塞等待
                if (how != ASYNC)
                    return awaitMatch(s, pred, e, (how == TIMED), nanos);
            }
            return e;
        }
    }

```

整个算法的核心就是寻找匹配节点找到了就返回，否则就入队（NOW直接返回）：

*   matched。判断匹配条件（isData不一样，本身没有匹配），匹配后就casItem，然后unpark匹配节点的waiter线程，如果是reservation则叫人收货，data则叫null收货。
*   unmatched。如果没有找到匹配节点，则根据传入的how来处理，NOW直接返回，其余三种先入对，入队后如果是ASYNC则返回，SYNC和TIMED则会阻塞等待匹配。

其实相当于SynchronousQueue来说，这个处理逻辑还是比较简单的。

如果没有找到匹配节点，且how != NOW会入队，入队则是调用tryAppend方法：

```
    private Node tryAppend(Node s, boolean haveData) {
        // 从尾节点tail开始
        for (Node t = tail, p = t;;) {
            Node n, u;

            // 队列为空则将节点S设置为head
            if (p == null && (p = head) == null) {
                if (casHead(null, s))
                    return s;
            }

            // 如果为data
            else if (p.cannotPrecede(haveData))
                return null;

            // 不是最后一个节点
            else if ((n = p.next) != null)
                p = p != t && t != (u = tail) ? (t = u) : (p != n) ? n : null;
            // CAS失败，一般来说失败的原因在于p.next != null，可能有其他增加了tail，向前推荐
            else if (!p.casNext(null, s))
                p = p.next;                   // re-read on CAS failure
            else {
                if (p != t) {                 // update if slack now >= 2
                    while ((tail != t || !casTail(t, s)) &&
                            (t = tail)   != null &&
                            (s = t.next) != null && // advance and retry
                            (s = s.next) != null && s != t);
                }
                return p;
            }
        }
    }

```

tryAppend方法是将S节点添加到tail上，然后返回其前驱节点。好吧，我承认这段代码我看的有点儿晕！！！

加入队列后，如果how还不是ASYNC则调用awaitMatch()方法阻塞等待：

```
    private E awaitMatch(Node s, Node pred, E e, boolean timed, long nanos) {
        // 超时控制
        final long deadline = timed ? System.nanoTime() + nanos : 0L;

        // 当前线程
        Thread w = Thread.currentThread();

        // 自旋次数
        int spins = -1; // initialized after first item and cancel checks

        // 随机数
        ThreadLocalRandom randomYields = null; // bound if needed

        for (;;) {
            Object item = s.item;
            //匹配了，可能有其他线程匹配了线程
            if (item != e) {
                // 撤销该节点
                s.forgetContents();
                return LinkedTransferQueue.<E>cast(item);
            }

            // 线程中断或者超时了。则调用将s节点item设置为e，等待取消
            if ((w.isInterrupted() || (timed && nanos <= 0)) && s.casItem(e, s)) {        // cancel
                // 断开节点
                unsplice(pred, s);
                return e;
            }

            // 自旋
            if (spins < 0) {
                // 计算自旋次数
                if ((spins = spinsFor(pred, s.isData)) > 0)
                    randomYields = ThreadLocalRandom.current();
            }

            // 自旋
            else if (spins > 0) {
                --spins;
                // 生成的随机数 == 0 ，停止线程？不是很明白....
                if (randomYields.nextInt(CHAINED_SPINS) == 0)
                    Thread.yield();
            }

            // 将当前线程设置到节点的waiter域
            // 一开始s.waiter == null 肯定是会成立的，
            else if (s.waiter == null) {
                s.waiter = w;                 // request unpark then recheck
            }

            // 超时阻塞
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos > 0L)
                    LockSupport.parkNanos(this, nanos);
            }
            else {
                // 不是超时阻塞
                LockSupport.park(this);
            }
        }
    }

```

整个awaitMatch过程和SynchronousQueue的awaitFulfill没有很大区别，不过在自旋过程会调用Thread.yield();这是干嘛？

在awaitMatch过程中，如果线程中断了，或者超时了则会调用unsplice()方法去除该节点：

```
    final void unsplice(Node pred, Node s) {
        s.forgetContents(); // forget unneeded fields

        if (pred != null && pred != s && pred.next == s) {
            Node n = s.next;
            if (n == null ||
                    (n != s && pred.casNext(s, n) && pred.isMatched())) {

                for (;;) {               // check if at, or could be, head
                    Node h = head;
                    if (h == pred || h == s || h == null)
                        return;          // at head or list empty
                    if (!h.isMatched())
                        break;
                    Node hn = h.next;
                    if (hn == null)
                        return;          // now empty
                    if (hn != h && casHead(h, hn))
                        h.forgetNext();  // advance head
                }
                if (pred.next != pred && s.next != s) { // recheck if offlist
                    for (;;) {           // sweep now if enough votes
                        int v = sweepVotes;
                        if (v < SWEEP_THRESHOLD) {
                            if (casSweepVotes(v, v + 1))
                                break;
                        }
                        else if (casSweepVotes(v, 0)) {
                            sweep();
                            break;
                        }
                    }
                }
            }
        }
    }

```

主体流程已经完成，这里总结下：

1.  无论是入对、出对，还是交换，最终都会跑到xfer(E e, boolean haveData, int how, long nanos)方法中，只不过传入的how不同而已
2.  如果队列不为空，则尝试在队列中寻找是否存在与该节点相匹配的节点，如果找到则将匹配节点的item设置e，然后唤醒匹配节点的waiter线程。如果是reservation则叫人收货，data则叫null收货
3.  如果队列为空，或者没有找到匹配的节点且how ！= NOW，则调用tryAppend()方法将节点添加到队列的tail，然后返回其前驱节点
4.  如果节点的how != NOW && how != ASYNC，则调用awaitMatch()方法阻塞等待，在阻塞等待过程中和SynchronousQuque的awaitFulfill()逻辑差不多，都是先自旋，然后判断是否需要自旋，如果中断或者超时了则将该节点从队列中移出

### 实例

这段摘自[JAVA 1.7并发之LinkedTransferQueue原理理解](http://www.cnblogs.com/rockman12352/p/3790245.html)。感觉看完上面的源码后，在结合这个例子会有更好的了解，掌握。

1：Head->Data Input->Data
Match: 根据他们的属性 发现 cannot match ，因为是同类的
处理节点: 所以把新的data放在原来的data后面，然后head往后移一位，Reservation同理
HEAD=DATA->DATA

2：Head->Data Input->Reservation （取数据）
Match: 成功match，就把Data的item变为reservation的值（null,有主了），并且返回数据。
处理节点： 没动，head还在原地
HEAD=DATA（用过）

3：Head->Reservation Input->Data（放数据）
Match: 成功match，就把Reservation的item变为Data的值（有主了），并且叫waiter来取
处理节点： 没动
HEAD=RESERVATION(用过)

  

##  总结

BlockingQueue

BlockingQueue接口实现Queue接口，它支持两个附加操作：获取元素时等待队列变为非空，以及存储元素时等待空间变得可用。相对于同一操作他提供了四种机制：抛出异常、返回特殊值、阻塞等待、超时：

![BlockingQueue操作](https://img-blog.csdn.net/20171004182403214?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

BlockingQueue常用于生产者和消费者场景。

JDK 8 中提供了七个阻塞队列可供使用（上图的DelayedWorkQueue是ScheduledThreadPoolExecutor的内部类）：

*   ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。
*   LinkedBlockingQueue ：一个由链表结构组成的无界阻塞队列。
*   PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。
*   DelayQueue：一个使用优先级队列实现的无界阻塞队列。
*   SynchronousQueue：一个不存储元素的阻塞队列。
*   LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
*   LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

### ArrayBlockingQueue

基于数组的阻塞队列，ArrayBlockingQueue内部维护这一个定长数组，阻塞队列的大小在初始化时就已经确定了，其后无法更改。

采用可重入锁ReentrantLock来保证线程安全性，但是生产者和消费者是共用同一个锁对象，这样势必会导致降低一定的吞吐量。当然ArrayBlockingQueue完全可以采用分离锁来实现生产者和消费者的并行操作，但是我认为这样做只会给代码带来额外的复杂性，对于性能而言应该不会有太大的提升，因为基于数组的ArrayBlockingQueue在数据的写入和读取操作已经非常轻巧了。

ArrayBlockingQueue支持公平性和非公平性，默认采用非公平模式，可以通过构造函数设置为公平访问策略（true）。

### PriorityBlockingQueue

PriorityBlockingQueue是支持优先级的无界队列。默认情况下采用自然顺序排序，当然也可以通过自定义Comparator来指定元素的排序顺序。

PriorityBlockingQueue内部采用二叉堆的实现方式，整个处理过程并不是特别复杂。添加操作则是不断“上冒”，而删除操作则是不断“下掉”。

### DelayQueue

DelayQueue是一个支持延时操作的无界阻塞队列。列头的元素是最先“到期”的元素，如果队列里面没有元素到期，是不能从列头获取元素的，哪怕有元素也不行。也就是说只有在延迟期满时才能够从队列中去元素。

它主要运用于如下场景：

*   缓存系统的设计：缓存是有一定的时效性的，可以用DelayQueue保存缓存的有效期，然后利用一个线程查询DelayQueue，如果取到元素就证明该缓存已经失效了。
*   定时任务的调度：DelayQueue保存当天将要执行的任务和执行时间，一旦取到元素（任务），就执行该任务。

DelayQueue采用支持优先级的PriorityQueue来实现，但是队列中的元素必须要实现Delayed接口，Delayed接口用来标记那些应该在给定延迟时间之后执行的对象，该接口提供了getDelay()方法返回元素节点的剩余时间。同时，元素也必须要实现compareTo()方法，compareTo()方法需要提供与getDelay()方法一致的排序。

### SynchronousQueue

SynchronousQueue是一个神奇的队列，他是一个不存储元素的阻塞队列，也就是说他的每一个put操作都需要等待一个take操作，否则就不能继续添加元素了，有点儿像[Exchanger](http://cmsblogs.com/?p=2269)，类似于生产者和消费者进行交换。

队列本身不存储任何元素，所以非常适用于传递性场景，两者直接进行对接。其吞吐量会高于ArrayBlockingQueue和LinkedBlockingQueue。

SynchronousQueue支持公平和非公平的访问策略，在默认情况下采用非公平性，也可以通过构造函数来设置为公平性。

SynchronousQueue的实现核心为Transferer接口，该接口有TransferQueue和TransferStack两个实现类，分别对应着公平策略和非公平策略。接口Transferer有一个tranfer()方法，该方法定义了转移数据，如果e != null，相当于将一个数据交给消费者，如果e == null，则相当于从一个生产者接收一个消费者交出的数据。

### LinkedTransferQueue

LinkedTransferQueue是一个由链表组成的的无界阻塞队列，该队列是一个相当牛逼的队列：它是ConcurrentLinkedQueue、SynchronousQueue (公平模式下)、无界的LinkedBlockingQueues等的超集。

与其他BlockingQueue相比，他多实现了一个接口TransferQueue，该接口是对BlockingQueue的一种补充，多了tryTranfer()和transfer()两类方法：

*   tranfer()：若当前存在一个正在等待获取的消费者线程，即立刻移交之。 否则，会插入当前元素e到队列尾部，并且等待进入阻塞状态，到有消费者线程取走该元素
*   tryTranfer()： 若当前存在一个正在等待获取的消费者线程（使用take()或者poll()函数），使用该方法会即刻转移/传输对象元素e；若不存在，则返回false，并且不进入队列。这是一个不阻塞的操作

### LinkedBlockingDeque

LinkedBlockingDeque是一个有链表组成的双向阻塞队列，与前面的阻塞队列相比它支持从两端插入和移出元素。以first结尾的表示从对头操作，以last结尾的表示从对尾操作。

在初始化LinkedBlockingDeque时可以初始化队列的容量，用来防止其再扩容时过渡膨胀。另外双向阻塞队列可以运用在“工作窃取”模式中。