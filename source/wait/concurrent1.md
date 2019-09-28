---
title: Java并发指南1：并发基础与Java多线程
date: 2018-05-16 23:05:11
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

# 什么是并发

在过去单CPU时代，单任务在一个时间点只能执行单一程序。之后发展到多任务阶段，计算机能在同一时间点并行执行多任务或多进程。虽然并不是真正意义上的“同一时间点”，而是多个任务或进程共享一个CPU，并交由操作系统来完成多任务间对CPU的运行切换，以使得每个任务都有机会获得一定的时间片运行。

随着多任务对软件开发者带来的新挑战，程序不在能假设独占所有的CPU时间、所有的内存和其他计算机资源。一个好的程序榜样是在其不再使用这些资源时对其进行释放，以使得其他程序能有机会使用这些资源。

再后来发展到多线程技术，使得在一个程序内部能拥有多个线程并行执行。一个线程的执行可以被认为是一个CPU在执行该程序。当一个程序运行在多线程下，就好像有多个CPU在同时执行该程序。

多线程比多任务更加有挑战。多线程是在同一个程序内部并行执行，因此会对相同的内存空间进行并发读写操作。这可能是在单线程程序中从来不会遇到的问题。其中的一些错误也未必会在单CPU机器上出现，因为两个线程从来不会得到真正的并行执行。然而，更现代的计算机伴随着多核CPU的出现，也就意味着不同的线程能被不同的CPU核得到真正意义的并行执行。

如果一个线程在读一个内存时，另一个线程正向该内存进行写操作，那进行读操作的那个线程将获得什么结果呢？是写操作之前旧的值？还是写操作成功之后的新值？或是一半新一半旧的值？或者，如果是两个线程同时写同一个内存，在操作完成后将会是什么结果呢？是第一个线程写入的值？还是第二个线程写入的值？还是两个线程写入的一个混合值？因此如没有合适的预防措施，任何结果都是可能的。而且这种行为的发生甚至不能预测，所以结果也是不确定性的。

## Java的多线程和并发性

Java是最先支持多线程的开发的语言之一，Java从一开始就支持了多线程能力，因此Java开发者能常遇到上面描述的问题场景。这也是我想为Java并发技术而写这篇系列的原因。作为对自己的笔记，和对其他Java开发的追随者都可获益的。

该系列主要关注Java多线程，但有些在多线程中出现的问题会和多任务以及分布式系统中出现的存在类似，因此该系列会将多任务和分布式系统方面作为参考，所以叫法上称为“并发性”，而不是“多线程”。

# 多线程的优点

尽管面临很多挑战，多线程有一些优点使得它一直被使用。这些优点是：





*   资源利用率更好
*   程序设计在某些情况下更简单
*   程序响应更快





## 资源利用率更好

想象一下，一个应用程序需要从本地文件系统中读取和处理文件的情景。比方说，从磁盘读取一个文件需要5秒，处理一个文件需要2秒。处理两个文件则需要：









| `1` | `5``秒读取文件A` |
| --- | --- |





| `2` | `2``秒处理文件A` |
| --- | --- |





| `3` | `5``秒读取文件B` |
| --- | --- |





| `4` | `2``秒处理文件B` |
| --- | --- |





| `5` | `---------------------` |
| --- | --- |





| `6` | `总共需要``14``秒` |
| --- | --- |







从磁盘中读取文件的时候，大部分的CPU时间用于等待磁盘去读取数据。在这段时间里，CPU非常的空闲。它可以做一些别的事情。通过改变操作的顺序，就能够更好的使用CPU资源。看下面的顺序：









| `1` | `5``秒读取文件A` |
| --- | --- |





| `2` | `5``秒读取文件B + ``2``秒处理文件A` |
| --- | --- |





| `3` | `2``秒处理文件B` |
| --- | --- |





| `4` | `---------------------` |
| --- | --- |





| `5` | `总共需要``12``秒` |
| --- | --- |









CPU等待第一个文件被读取完。然后开始读取第二个文件。当第二文件在被读取的时候，CPU会去处理第一个文件。记住，在等待磁盘读取文件的时候，CPU大部分时间是空闲的。



总的说来，CPU能够在等待IO的时候做一些其他的事情。这个不一定就是磁盘IO。它也可以是网络的IO，或者用户输入。通常情况下，网络和磁盘的IO比CPU和内存的IO慢的多。







## 程序设计更简单



在单线程应用程序中，如果你想编写程序手动处理上面所提到的读取和处理的顺序，你必须记录每个文件读取和处理的状态。相反，你可以启动两个线程，每个线程处理一个文件的读取和操作。线程会在等待磁盘读取文件的过程中被阻塞。在等待的时候，其他的线程能够使用CPU去处理已经读取完的文件。其结果就是，磁盘总是在繁忙地读取不同的文件到内存中。这会带来磁盘和CPU利用率的提升。而且每个线程只需要记录一个文件，因此这种方式也很容易编程实现。



## 程序响应更快

将一个单线程应用程序变成多线程应用程序的另一个常见的目的是实现一个响应更快的应用程序。设想一个服务器应用，它在某一个端口监听进来的请求。当一个请求到来时，它去处理这个请求，然后再返回去监听。



服务器的流程如下所述：







| `1` | `while``(server is active){` |
| --- | --- |





| `2` | `listen ``for` `request` |
| --- | --- |





| `3` | `process request` |
| --- | --- |





| `4` | `}` |
| --- | --- |









如果一个请求需要占用大量的时间来处理，在这段时间内新的客户端就无法发送请求给服务端。只有服务器在监听的时候，请求才能被接收。另一种设计是，监听线程把请求传递给工作者线程(worker thread)，然后立刻返回去监听。而工作者线程则能够处理这个请求并发送一个回复给客户端。这种设计如下所述：









| `1` | `while``(server is active){` |
| --- | --- |





| `2` | `listen ``for` `request` |
| --- | --- |





| `3` | `hand request to worker thread` |
| --- | --- |





| `4` | `}` |
| --- | --- |









这种方式，服务端线程迅速地返回去监听。因此，更多的客户端能够发送请求给服务端。这个服务也变得响应更快。



桌面应用也是同样如此。如果你点击一个按钮开始运行一个耗时的任务，这个线程既要执行任务又要更新窗口和按钮，那么在任务执行的过程中，这个应用程序看起来好像没有反应一样。相反，任务可以传递给工作者线程（word thread)。当工作者线程在繁忙地处理任务的时候，窗口线程可以自由地响应其他用户的请求。当工作者线程完成任务的时候，它发送信号给窗口线程。窗口线程便可以更新应用程序窗口，并显示任务的结果。对用户而言，这种具有工作者线程设计的程序显得响应速度更快。





















# 多线程的代价

从一个单线程的应用到一个多线程的应用并不仅仅带来好处，它也会有一些代价。不要仅仅为了使用多线程而使用多线程。而应该明确在使用多线程时能多来的好处比所付出的代价大的时候，才使用多线程。如果存在疑问，应该尝试测量一下应用程序的性能和响应能力，而不只是猜测。



## 设计更复杂



虽然有一些多线程应用程序比单线程的应用程序要简单，但其他的一般都更复杂。在多线程访问共享数据的时候，这部分代码需要特别的注意。线程之间的交互往往非常复杂。不正确的线程同步产生的错误非常难以被发现，并且重现以修复。



## 上下文切换的开销



当CPU从执行一个线程切换到执行另外一个线程的时候，它需要先存储当前线程的本地的数据，程序指针等，然后载入另一个线程的本地数据，程序指针等，最后才开始执行。这种切换称为“上下文切换”(“context switch”)。CPU会在一个上下文中执行一个线程，然后切换到另外一个上下文中执行另外一个线程。

上下文切换并不廉价。如果没有必要，应该减少上下文切换的发生。

你可以通过维基百科阅读更多的关于上下文切换相关的内容：

[http://en.wikipedia.org/wiki/Context_switch](http://en.wikipedia.org/wiki/Context_switch)



## 增加资源消耗



线程在运行的时候需要从计算机里面得到一些资源。除了CPU，线程还需要一些内存来维持它本地的堆栈。它也需要占用操作系统中一些资源来管理线程。我们可以尝试编写一个程序，让它创建100个线程，这些线程什么事情都不做，只是在等待，然后看看这个程序在运行的时候占用了多少内存。


## 竞态条件与临界区



在同一程序中运行多个线程本身不会导致问题，问题在于多个线程访问了相同的资源。如，同一内存区（变量，数组，或对象）、系统（数据库，web services等）或文件。实际上，这些问题只有在一或多个线程向这些资源做了写操作时才有可能发生，只要资源没有发生变化,多个线程读取相同的资源就是安全的。



多线程同时执行下面的代码可能会出错：







| `1` | `public` `class` `Counter {` |
| --- | --- |





| `2` | `protected` `long` `count = ``0``;` |
| --- | --- |





| `3` | `public` `void` `add(``long` `value){` |
| --- | --- |





| `4` | `this``.count = ``this``.count + value;  ` |
| --- | --- |





| `5` | `}` |
| --- | --- |





| `6` | `}` |
| --- | --- |







想象下线程A和B同时执行同一个Counter对象的add()方法，我们无法知道操作系统何时会在两个线程之间切换。JVM并不是将这段代码视为单条指令来执行的，而是按照下面的顺序：

<pre>从内存获取 this.count 的值放到寄存器
将寄存器中的值增加value
将寄存器中的值写回内存
</pre>

观察线程A和B交错执行会发生什么：

<pre>	this.count = 0;
   A:	读取 this.count 到一个寄存器 (0)
   B:	读取 this.count 到一个寄存器 (0)
   B: 	将寄存器的值加2
   B:	回写寄存器值(2)到内存. this.count 现在等于 2
   A:	将寄存器的值加3
   A:	回写寄存器值(3)到内存. this.count 现在等于 3
</pre>

两个线程分别加了2和3到count变量上，两个线程执行结束后count变量的值应该等于5。然而由于两个线程是交叉执行的，两个线程从内存中读出的初始值都是0。然后各自加了2和3，并分别写回内存。最终的值并不是期望的5，而是最后写回内存的那个线程的值，上面例子中最后写回内存的是线程A，但实际中也可能是线程B。如果没有采用合适的同步机制，线程间的交叉执行情况就无法预料。


## 竞态条件&临界区


当两个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。导致竞态条件发生的代码区称作临界区。上例中add()方法就是一个临界区,它会产生竞态条件。在临界区中使用适当的同步就可以避免竞态条件。

# 线程安全与共享资源

允许被多个线程同时执行的代码称作线程安全的代码。线程安全的代码不包含竞态条件。当多个线程同时更新共享资源时会引发竞态条件。因此，了解Java线程执行时共享了什么资源很重要。



## 局部变量

局部变量存储在线程自己的栈中。也就是说，局部变量永远也不会被多个线程共享。所以，基础类型的局部变量是线程安全的。下面是基础类型的局部变量的一个例子：







| `1` | `public` `void` `someMethod(){` |
| --- | --- |





| `2` |   |
| --- | --- |





| `3` | `long` `threadSafeInt = ``0``;` |
| --- | --- |





| `4` |   |
| --- | --- |





| `5` | `threadSafeInt++;` |
| --- | --- |





| `6` | `}` |
| --- | --- |







## 局部的对象引用

对象的局部引用和基础类型的局部变量不太一样。尽管引用本身没有被共享，但引用所指的对象并没有存储在线程的栈内。所有的对象都存在共享堆中。如果在某个方法中创建的对象不会逃逸出（_译者注：即该对象不会被其它方法获得，也不会被非局部变量引用到_）该方法，那么它就是线程安全的。实际上，哪怕将这个对象作为参数传给其它方法，只要别的线程获取不到这个对象，那它仍是线程安全的。下面是一个线程安全的局部引用样例：







| `01` | `public` `void` `someMethod(){` |
| --- | --- |





| `02` |   |
| --- | --- |





| `03` | `LocalObject localObject = ``new` `LocalObject();` |
| --- | --- |





| `04` |   |
| --- | --- |





| `05` | `localObject.callMethod();` |
| --- | --- |





| `06` | `method2(localObject);` |
| --- | --- |





| `07` | `}` |
| --- | --- |





| `08` |   |
| --- | --- |





| `09` | `public` `void` `method2(LocalObject localObject){` |
| --- | --- |





| `10` | `localObject.setValue(``"value"``);` |
| --- | --- |





| `11` | `}` |
| --- | --- |







样例中LocalObject对象没有被方法返回，也没有被传递给someMethod()方法外的对象。每个执行someMethod()的线程都会创建自己的LocalObject对象，并赋值给localObject引用。因此，这里的LocalObject是线程安全的。事实上，整个someMethod()都是线程安全的。即使将LocalObject作为参数传给同一个类的其它方法或其它类的方法时，它仍然是线程安全的。当然，如果LocalObject通过某些方法被传给了别的线程，那它就不再是线程安全的了。

## 对象成员

对象成员存储在堆上。如果两个线程同时更新同一个对象的同一个成员，那这个代码就不是线程安全的。下面是一个样例：







| `1` | `public` `class` `NotThreadSafe{` |
| --- | --- |





| `2` | `StringBuilder builder = ``new` `StringBuilder();` |
| --- | --- |





| `3` |   |
| --- | --- |





| `4` | `public` `add(String text){` |
| --- | --- |





| `5` | `this``.builder.append(text);` |
| --- | --- |





| `6` | `}  ` |
| --- | --- |





| `7` | `}` |
| --- | --- |







如果两个线程同时调用同一个`NotThreadSafe`实例上的add()方法，就会有竞态条件问题。例如：







| `01` | `NotThreadSafe sharedInstance = ``new` `NotThreadSafe();` |
| --- | --- |





| `02` |   |
| --- | --- |





| `03` | `new` `Thread(``new` `MyRunnable(sharedInstance)).start();` |
| --- | --- |





| `04` | `new` `Thread(``new` `MyRunnable(sharedInstance)).start();` |
| --- | --- |





| `05` |   |
| --- | --- |





| `06` | `public` `class` `MyRunnable ``implements` `Runnable{` |
| --- | --- |





| `07` | `NotThreadSafe instance = ``null``;` |
| --- | --- |





| `08` |   |
| --- | --- |





| `09` | `public` `MyRunnable(NotThreadSafe instance){` |
| --- | --- |





| `10` | `this``.instance = instance;` |
| --- | --- |





| `11` | `}` |
| --- | --- |





| `12` |   |
| --- | --- |





| `13` | `public` `void` `run(){` |
| --- | --- |





| `14` | `this``.instance.add(``"some text"``);` |
| --- | --- |





| `15` | `}` |
| --- | --- |





| `16` | `}` |
| --- | --- |







注意两个MyRunnable共享了同一个NotThreadSafe对象。因此，当它们调用add()方法时会造成竞态条件。

当然，如果这两个线程在不同的NotThreadSafe实例上调用call()方法，就不会导致竞态条件。下面是稍微修改后的例子：







| `1` | `new` `Thread(``new` `MyRunnable(``new` `NotThreadSafe())).start();` |
| --- | --- |





| `2` | `new` `Thread(``new` `MyRunnable(``new` `NotThreadSafe())).start();` |
| --- | --- |







现在两个线程都有自己单独的NotThreadSafe对象，调用add()方法时就会互不干扰，再也不会有竞态条件问题了。所以非线程安全的对象仍可以通过某种方式来消除竞态条件。

## 线程控制逃逸规则

线程控制逃逸规则可以帮助你判断代码中对某些资源的访问是否是线程安全的。

<pre>如果一个资源的创建，使用，销毁都在同一个线程内完成，
且永远不会脱离该线程的控制，则该资源的使用就是线程安全的。
</pre>

资源可以是对象，数组，文件，数据库连接，套接字等等。Java中你无需主动销毁对象，所以“销毁”指不再有引用指向对象。

即使对象本身线程安全，但如果该对象中包含其他资源（文件，数据库连接），整个应用也许就不再是线程安全的了。比如2个线程都创建了各自的数据库连接，每个连接自身是线程安全的，但它们所连接到的同一个数据库也许不是线程安全的。比如，2个线程执行如下代码：

<pre>检查记录X是否存在，如果不存在，插入X
</pre>

如果两个线程同时执行，而且碰巧检查的是同一个记录，那么两个线程最终可能都插入了记录：

<pre>线程1检查记录X是否存在。检查结果：不存在
线程2检查记录X是否存在。检查结果：不存在
线程1插入记录X
线程2插入记录X
</pre>

同样的问题也会发生在文件或其他共享资源上。因此，区分某个线程控制的对象是资源本身，还是仅仅到某个资源的引用很重要。

# 线程安全及不可变性

当多个线程同时访问同一个资源，并且其中的一个或者多个线程对这个资源进行了写操作，才会产生**竞态条件**。多个线程同时读同一个资源不会产生竞态条件。

我们可以通过创建不可变的共享对象来保证对象在线程间共享时不会被修改，从而实现线程安全。如下示例：







| `01` | `public` `class` `ImmutableValue{` |
| --- | --- |





| `02` | `private` `int` `value = ``0``;` |
| --- | --- |





| `03` |   |
| --- | --- |





| `04` | `public` `ImmutableValue(``int` `value){` |
| --- | --- |





| `05` | `this``.value = value;` |
| --- | --- |





| `06` | `}` |
| --- | --- |





| `07` |   |
| --- | --- |





| `08` | `public` `int` `getValue(){` |
| --- | --- |





| `09` | `return` `this``.value;` |
| --- | --- |





| `10` | `}` |
| --- | --- |





| `11` | `}` |
| --- | --- |







请注意ImmutableValue类的成员变量`value`是通过构造函数赋值的，并且在类中没有set方法。这意味着一旦ImmutableValue实例被创建，`value`变量就不能再被修改，这就是不可变性。但你可以通过getValue()方法读取这个变量的值。

（_译者注：注意，“不变”（Immutable）和“只读”（Read Only）是不同的。当一个变量是“只读”时，变量的值不能直接改变，但是可以在其它变量发生改变的时候发生改变。比如，一个人的出生年月日是“不变”属性，而一个人的年龄便是“只读”属性，但是不是“不变”属性。随着时间的变化，一个人的年龄会随之发生变化，而一个人的出生年月日则不会变化。这就是“不变”和“只读”的区别。（摘自《Java与模式》第34章）_）

如果你需要对ImmutableValue类的实例进行操作，可以通过得到value变量后创建一个新的实例来实现，下面是一个对value变量进行加法操作的示例：







| `01` | `public` `class` `ImmutableValue{` |
| --- | --- |





| `02` | `private` `int` `value = ``0``;` |
| --- | --- |





| `03` |   |
| --- | --- |





| `04` | `public` `ImmutableValue(``int` `value){` |
| --- | --- |





| `05` | `this``.value = value;` |
| --- | --- |





| `06` | `}` |
| --- | --- |





| `07` |   |
| --- | --- |





| `08` | `public` `int` `getValue(){` |
| --- | --- |





| `09` | `return` `this``.value;` |
| --- | --- |





| `10` | `}` |
| --- | --- |





| `11` |   |
| --- | --- |





| `12` | `public` `ImmutableValue add(``int` `valueToAdd){` |
| --- | --- |





| `13` | `return` `new` `ImmutableValue(``this``.value + valueToAdd);` |
| --- | --- |





| `14` | `}` |
| --- | --- |





| `15` | `}` |
| --- | --- |







请注意add()方法以加法操作的结果作为一个新的ImmutableValue类实例返回，而不是直接对它自己的value变量进行操作。

## 引用不是线程安全的！

重要的是要记住，即使一个对象是线程安全的不可变对象，指向这个对象的引用也可能不是线程安全的。看这个例子：







| `01` | `public` `void` `Calculator{` |
| --- | --- |





| `02` | `private` `ImmutableValue currentValue = ``null``;` |
| --- | --- |





| `03` |   |
| --- | --- |





| `04` | `public` `ImmutableValue getValue(){` |
| --- | --- |





| `05` | `return` `currentValue;` |
| --- | --- |





| `06` | `}` |
| --- | --- |





| `07` |   |
| --- | --- |





| `08` | `public` `void` `setValue(ImmutableValue newValue){` |
| --- | --- |





| `09` | `this``.currentValue = newValue;` |
| --- | --- |





| `10` | `}` |
| --- | --- |





| `11` |   |
| --- | --- |





| `12` | `public` `void` `add(``int` `newValue){` |
| --- | --- |





| `13` | `this``.currentValue = ``this``.currentValue.add(newValue);` |
| --- | --- |





| `14` | `}` |
| --- | --- |





| `15` | `}` |
| --- | --- |







Calculator类持有一个指向ImmutableValue实例的引用。注意，通过setValue()方法和add()方法可能会改变这个引用。因此，即使Calculator类内部使用了一个不可变对象，但Calculator类本身还是可变的，因此Calculator类不是线程安全的。换句话说：ImmutableValue类是线程安全的，但使用它的类不是。当尝试通过不可变性去获得线程安全时，这点是需要牢记的。

要使Calculator类实现线程安全，将getValue()、setValue()和add()方法都声明为同步方法即可。


# Java多线程基础

## 1 线程与多线程

线程是什么？
线程（Thread）是一个对象（Object）。用来干什么？Java 线程（也称 JVM 线程）是 Java 进程内允许多个同时进行的任务。该进程内并发的任务成为线程（Thread），一个进程里至少一个线程。

Java 程序采用多线程方式来支持大量的并发请求处理，程序如果在多线程方式执行下，其复杂度远高于单线程串行执行。那么多线程：指的是这个程序（一个进程）运行时产生了不止一个线程。

为啥使用多线程？

*   适合多核处理器。一个线程运行在一个处理器核心上，那么多线程可以分配到多个处理器核心上，更好地利用多核处理器。
*   防止阻塞。将数据一致性不强的操作使用多线程技术（或者消息队列）加快代码逻辑处理，缩短响应时间。

聊到多线程，多半会聊并发与并行，咋理解并区分这两个的区别呢？

*   类似单个 CPU ，通过 CPU 调度算法等，处理多个任务的能力，叫并发
*   类似多个 CPU ，同时并且处理相同多个任务的能力，叫做并行

## 2 线程的运行与创建

### 2.1 线程的创建

Java 创建线程对象有两种方法：

*   继承 Thread 类创建线程对象
*   实现 Runnable 接口类创建线程对象

新建 MyThread 对象，代码如下：

```
/**
 * 继承 Thread 类创建线程对象
 * @author Jeff Lee @ bysocket.com
 * @since 2018年01月27日21:03:02
 */
public class MyThread extends Thread {

    @Override // 可以省略
    public void run() {
        System.out.println("MyThread 的线程对象正在执行任务");
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            MyThread thread = new MyThread();
            thread.start();

            System.out.println("MyThread 的线程对象 " + thread.getId());
        }
    }
}

```

MyThread 类继承了 Thread 对象，并重写（Override）了 run 方法，实现线程里面的逻辑。main 函数是使用 for 语句，循环创建了 10 个线程，调用 start 方法启动线程，最后打印当前线程对象的 ID。

run 方法和 start 方法的区别是什么呢？
run 方法就是跑的意思，线程启动后，会调用 run 方法。
start 方法就是启动的意思，就是启动新线程实例。启动线程后，才会调线程的 run 方法。

执行 main 方法后，控制台打印如下：

```
MyThread 的线程对象正在执行任务
MyThread 的线程对象 10
MyThread 的线程对象正在执行任务
MyThread 的线程对象 11
MyThread 的线程对象正在执行任务
MyThread 的线程对象 12
MyThread 的线程对象正在执行任务
MyThread 的线程对象 13
MyThread 的线程对象正在执行任务
MyThread 的线程对象 14
MyThread 的线程对象正在执行任务
MyThread 的线程对象 15
MyThread 的线程对象正在执行任务
MyThread 的线程对象 16
MyThread 的线程对象正在执行任务
MyThread 的线程对象 17
MyThread 的线程对象正在执行任务
MyThread 的线程对象 18
MyThread 的线程对象正在执行任务
MyThread 的线程对象 19

```

可见，线程的 ID 是线程唯一标识符，每个线程 ID 都是不一样的。

start 方法和 run 方法的关系如图所示：
![](http://upload-images.jianshu.io/upload_images/1483536-c52b04e3ae79a99a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同理，实现 Runnable 接口类创建线程对象也很简单，只是不同的形式。新建 MyThreadBrother 代码如下：

```
/**
 * 实现 Runnable 接口类创建线程对象
 * @author Jeff Lee @ bysocket.com
 * @since 2018年01月27日21:22:57
 */
public class MyThreadBrother implements Runnable {

    @Override // 可以省略
    public void run() {
        System.out.println("MyThreadBrother 的线程对象正在执行任务");
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            Thread thread = new Thread(new MyThreadBrother());
            thread.start();

            System.out.println("MyThreadBrother 的线程对象 " + thread.getId());
        }
    }
}

```

具体代码：「java-concurrency-core-learning」
[https://github.com/JeffLi1993/java-concurrency-core-learning](https://github.com/JeffLi1993/java-concurrency-core-learning)

### 2.1 线程的运行

在运行上面两个小 demo 后，JVM 执行了 main 函数线程，然后在主线程中执行创建了新的线程。正常情况下，所有线程执行到运行结束为止。除非某个线程中调用了 System.exit(1) 则被终止。

在实际开发中，一个请求到响应式是一个线程。但在这个线程中可以使用线程池创建新的线程，去执行任务。

![](http://upload-images.jianshu.io/upload_images/1483536-4c54f3f145fb77c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3 线程的状态

新建 MyThreadInfo 类，打印线程对象属性，代码如下：

```
/**
 * 线程实例对象的属性值
 * @author Jeff Lee @ bysocket.com
 * @since 2018年01月27日21:24:40
 */
public class MyThreadInfo extends Thread {

    @Override // 可以省略
    public void run() {
        System.out.println("MyThreadInfo 的线程实例正在执行任务");
//        System.exit(1);
    }

    public static void main(String[] args) {
        MyThreadInfo thread = new MyThreadInfo();
        thread.start();

        System.out.print("MyThreadInfo 的线程对象 \n"
                + "线程唯一标识符：" + thread.getId() + "\n"
                + "线程名称：" + thread.getName() + "\n"
                + "线程状态：" + thread.getState() + "\n"
                + "线程优先级：" + thread.getPriority());
    }
}

```

执行代码打印如下：

```
MyThreadInfo 的线程实例正在执行任务
MyThreadInfo 的线程对象 
线程唯一标识符：10
线程名称：Thread-0
线程状态：NEW
线程优先级：5

```

线程是一个对象，它有唯一标识符 ID、名称、状态、优先级等属性。线程只能修改其优先级和名称等属性 ，无法修改 ID 、状态。ID 是 JVM 分配的，名字默认也为 Thread-XX，XX是一组数字。线程初始状态为 NEW。

线程优先级的范围是 1 到 10 ，其中 1 是最低优先级，10 是最高优先级。不推荐改变线程的优先级，如果业务需要，自然可以修改线程优先级到最高，或者最低。

线程的状态实现通过 Thread.State 常量类实现，有 6 种线程状态：new（新建）、runnnable（可运行）、blocked（阻塞）、waiting（等待）、time waiting （定时等待）和 terminated（终止）。状态转换图如下：

![](http://upload-images.jianshu.io/upload_images/1483536-d8214781072e129f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

线程状态流程大致如下：

*   线程创建后，进入 new 状态
*   调用 start 或者 run 方法，进入 runnable 状态
*   JVM 按照线程优先级及时间分片等执行 runnable 状态的线程。开始执行时，进入 running 状态
*   如果线程执行 sleep、wait、join，或者进入 IO 阻塞等。进入 wait 或者 blocked 状态
*   线程执行完毕后，线程被线程队列移除。最后为 terminated 状态。

## 4 小结

本文介绍了线程与多线程的基础篇，包括了线程启动及线程状态等。下一篇我们聊下线程的具体操作。包括中断、终止等

## 线程中断和终止

### 1 线程中断

#### 1.1 什么是线程中断？

线程中断是线程的标志位属性。而不是真正终止线程，和线程的状态无关。线程中断过程表示一个运行中的线程，通过其他线程调用了该线程的 `interrupt()` 方法，使得该线程中断标志位属性改变。

深入思考下，线程中断不是去中断了线程，恰恰是用来通知该线程应该被中断了。具体是一个标志位属性，到底该线程生命周期是去终止，还是继续运行，由线程根据标志位属性自行处理。

#### 1.2 线程中断操作

调用线程的 `interrupt()` 方法，根据线程不同的状态会有不同的结果。

下面新建 InterruptedThread 对象，代码如下：

```
/**
 * 一直运行的线程，中断状态为 true
 *
 * @author Jeff Lee @ bysocket.com
 * @since 2018年02月23日19:03:02
 */
public class InterruptedThread implements Runnable {

    @Override // 可以省略
    public void run() {
        // 一直 run
        while (true) {
        }
    }

    public static void main(String[] args) throws Exception {

        Thread interruptedThread = new Thread(new InterruptedThread(), "InterruptedThread");
        interruptedThread.start();

        TimeUnit.SECONDS.sleep(2);

        interruptedThread.interrupt();
        System.out.println("InterruptedThread interrupted is " + interruptedThread.isInterrupted());

        TimeUnit.SECONDS.sleep(2);
    }
}

```

运行 main 函数，结果如下：

```
InterruptedThread interrupted is true

```

代码详解：

*   线程一直在运行状态，没有停止或者阻塞等
*   调用了 `interrupt()` 方法，中断状态置为 true，但不会影响线程的继续运行

另一种情况，新建 InterruptedException 对象，代码如下：

```
/**
 * 抛出 InterruptedException 的线程，中断状态被重置为默认状态 false
 *
 * @author Jeff Lee @ bysocket.com
 * @since 2018年02月23日19:03:02
 */
public class InterruptedException implements Runnable {

    @Override // 可以省略
    public void run() {
        // 一直 sleep
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (java.lang.InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception {

        Thread interruptedThread = new Thread(new InterruptedException(), "InterruptedThread");
        interruptedThread.start();

        TimeUnit.SECONDS.sleep(2);

        // 中断被阻塞状态（sleep、wait、join 等状态）的线程，会抛出异常 InterruptedException
        // 在抛出异常 InterruptedException 前，JVM 会先将中断状态重置为默认状态 false
        interruptedThread.interrupt();
        System.out.println("InterruptedThread interrupted is " + interruptedThread.isInterrupted());
        TimeUnit.SECONDS.sleep(2);
    }
}

```

运行 main 函数，结果如下：

```
InterruptedThread interrupted is false
java.lang.InterruptedException: sleep interrupted
    at java.lang.Thread.sleep(Native Method)

```

代码详解：

*   中断被阻塞状态（sleep、wait、join 等状态）的线程，会抛出异常 InterruptedException
*   抛出异常 InterruptedException 前，JVM 会先将中断状态重置为默认状态 false

小结下线程中断：

*   线程中断，不是停止线程，只是一个线程的标志位属性
*   如果线程状态为被阻塞状态（sleep、wait、join 等状态），线程状态退出被阻塞状态，抛出异常 InterruptedException，并重置中断状态为默认状态 false
*   如果线程状态为运行状态，线程状态不变，继续运行，中断状态置为 true

代码：[https://github.com/JeffLi1993/java-concurrency-core-learning](https://github.com/JeffLi1993/java-concurrency-core-learning)

### 2 线程终止

比如在 IDEA 中强制关闭程序，立即停止程序，不给程序释放资源等操作，肯定是不正确的。线程终止也存在类似的问题，所以需要考虑如何终止线程？

上面聊到了线程中断，可以利用线程中断标志位属性来安全终止线程。同理也可以使用 boolean 变量来控制是否需要终止线程。

新建 ，代码如下：

```
/**
 * 安全终止线程
 *
 * @author Jeff Lee @ bysocket.com
 * @since 2018年02月23日19:03:02
 */
public class ThreadSafeStop {

    public static void main(String[] args) throws Exception {
        Runner one = new Runner();
        Thread countThread = new Thread(one, "CountThread");
        countThread.start();
        // 睡眠 1 秒，通知 CountThread 中断，并终止线程
        TimeUnit.SECONDS.sleep(1);
        countThread.interrupt();

        Runner two = new Runner();
        countThread = new Thread(two,"CountThread");
        countThread.start();
        // 睡眠 1 秒，然后设置线程停止状态，并终止线程
        TimeUnit.SECONDS.sleep(1);
        two.stopSafely();
    }

    private static class Runner implements Runnable {

        private long i;

        // 终止状态
        private volatile boolean on = true;

        @Override
        public void run() {
            while (on && !Thread.currentThread().isInterrupted()) {
                // 线程执行具体逻辑
                i++;
            }
            System.out.println("Count i = " + i);
        }

        public void stopSafely() {
            on = false;
        }
    }
}

```

从上面代码可以看出，通过 `while (on && !Thread.currentThread().isInterrupted())` 代码来实现线程是否跳出执行逻辑，并终止。但是疑问点就来了，为啥需要 `on` 和 `isInterrupted()` 两项一起呢？用其中一个方式不就行了吗？答案在下面

*   线程成员变量 `on` 通过 volatile 关键字修饰，达到线程之间可见，从而实现线程的终止。但当线程状态为被阻塞状态（sleep、wait、join 等状态）时，对成员变量操作也阻塞，进而无法执行安全终止线程
*   为了处理上面的问题，引入了 `isInterrupted();` 只去解决阻塞状态下的线程安全终止。
*   两者结合是真的没问题了吗？不是的，如果是网络 io 阻塞，比如一个 websocket 一直再等待响应，那么直接使用底层的 close 。

### 3 小结

很多好友介绍，如果用 Spring 栈开发到使用线程或者线程池，那么尽量使用框架这块提供的线程操作及框架提供的终止等

# **Threadlocal介绍**

> 原文出处[http://cmsblogs.com/](http://cmsblogs.com/) 『chenssy』

## ThreadLoacal是什么？

ThreadLocal是啥？以前面试别人时就喜欢问这个，有些伙伴喜欢把它和线程同步机制混为一谈，事实上ThreadLocal与线程同步无关。ThreadLocal虽然提供了一种解决多线程环境下成员变量的问题，但是它并不是解决多线程共享变量的问题。那么ThreadLocal到底是什么呢？

API是这样介绍它的：
This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its {@code get} or {@code set} method) has its own, independently initialized copy of the variable. {@code ThreadLocal} instances are typically private static fields in classes that wish to associate state with a thread (e.g.,a user ID or Transaction ID).

> 该类提供了线程局部 (thread-local) 变量。这些变量不同于它们的普通对应物，因为访问某个变量（通过其`get` 或 `set` 方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。`ThreadLocal`实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。

所以ThreadLocal与线程同步机制不同，线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每一个线程创建一个单独的变量副本，故而每个线程都可以独立地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本。可以说ThreadLocal为多线程环境下变量问题提供了另外一种解决思路。

ThreadLocal定义了四个方法：

*   get()：返回此线程局部变量的当前线程副本中的值。
*   initialValue()：返回此线程局部变量的当前线程的“初始值”。
*   remove()：移除此线程局部变量当前线程的值。
*   set(T value)：将此线程局部变量的当前线程副本中的值设置为指定值。

除了这四个方法，ThreadLocal内部还有一个静态内部类ThreadLocalMap，该内部类才是实现线程隔离机制的关键，get()、set()、remove()都是基于该内部类操作。ThreadLocalMap提供了一种用键值对方式存储每一个线程的变量副本的方法，key为当前ThreadLocal对象，value则是对应线程的变量副本。

对于ThreadLocal需要注意的有两点：
1\. ThreadLocal实例本身是不存储值，它只是提供了一个在当前线程中找到副本值得key。
2\. 是ThreadLocal包含在Thread中，而不是Thread包含在ThreadLocal中，有些小伙伴会弄错他们的关系。

下图是Thread、ThreadLocal、ThreadLocalMap的关系（[http://blog.xiaohansong.com/2016/08/06/ThreadLocal-memory-leak/](http://blog.xiaohansong.com/2016/08/06/ThreadLocal-memory-leak/)）

![Thread、ThreadLocal、ThreadLocalMap的关系](https://img-blog.csdn.net/20171005154736367?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## ThreadLocal使用示例

示例如下：

```
public class SeqCount {

    private static ThreadLocal<Integer> seqCount = new ThreadLocal<Integer>(){
        // 实现initialValue()
        public Integer initialValue() {
            return 0;
        }
    };

    public int nextSeq(){
        seqCount.set(seqCount.get() + 1);

        return seqCount.get();
    }

    public static void main(String[] args){
        SeqCount seqCount = new SeqCount();

        SeqThread thread1 = new SeqThread(seqCount);
        SeqThread thread2 = new SeqThread(seqCount);
        SeqThread thread3 = new SeqThread(seqCount);
        SeqThread thread4 = new SeqThread(seqCount);

        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
    }

    private static class SeqThread extends Thread{
        private SeqCount seqCount;

        SeqThread(SeqCount seqCount){
            this.seqCount = seqCount;
        }

        public void run() {
            for(int i = 0 ; i < 3 ; i++){
                System.out.println(Thread.currentThread().getName() + " seqCount :" + seqCount.nextSeq());
            }
        }
    }
}

```

运行结果：

![运行结果](https://img-blog.csdn.net/20171005154820871?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从运行结果可以看出，ThreadLocal确实是可以达到线程隔离机制，确保变量的安全性。这里我们想一个问题，在上面的代码中ThreadLocal的initialValue()方法返回的是0，加入该方法返回得是一个对象呢，会产生什么后果呢？例如：

```
    A a = new A();
    private static ThreadLocal<A> seqCount = new ThreadLocal<A>(){
        // 实现initialValue()
        public A initialValue() {
            return a;
        }
    };

    class A{
        // ....
    }

```

具体过程请参考：[对ThreadLocal实现原理的一点思考](http://www.jianshu.com/p/ee8c9dccc953)

## ThreadLocal源码解析

ThreadLocal虽然解决了这个多线程变量的复杂问题，但是它的源码实现却是比较简单的。ThreadLocalMap是实现ThreadLocal的关键，我们先从它入手。

### ThreadLocalMap

ThreadLocalMap其内部利用Entry来实现key-value的存储，如下：

```
       static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

```

从上面代码中可以看出Entry的key就是ThreadLocal，而value就是值。同时，Entry也继承WeakReference，所以说Entry所对应key（ThreadLocal实例）的引用为一个弱引用（关于弱引用这里就不多说了，感兴趣的可以关注这篇博客：[Java 理论与实践: 用弱引用堵住内存泄漏](https://www.ibm.com/developerworks/cn/java/j-jtp11225/)）

ThreadLocalMap的源码稍微多了点，我们就看两个最核心的方法getEntry()、set(ThreadLocal> key, Object value)方法。

**set(ThreadLocal> key, Object value)**

```
    private void set(ThreadLocal<?> key, Object value) {

        ThreadLocal.ThreadLocalMap.Entry[] tab = table;
        int len = tab.length;

        // 根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
        int i = key.threadLocalHashCode & (len-1);

        // 采用“线性探测法”，寻找合适位置
        for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {

            ThreadLocal<?> k = e.get();

            // key 存在，直接覆盖
            if (k == key) {
                e.value = value;
                return;
            }

            // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
            if (k == null) {
                // 用新元素替换陈旧的元素
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        // ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
        tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);

        int sz = ++size;

        // cleanSomeSlots 清楚陈旧的Entry（key == null）
        // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }

```

这个set()操作和我们在集合了解的put()方式有点儿不一样，虽然他们都是key-value结构，不同在于他们解决散列冲突的方式不同。集合Map的put()采用的是拉链法，而ThreadLocalMap的set()则是采用开放定址法（具体请参考[散列冲突处理系列博客](http://www.nowamagic.net/academy/detail/3008015)）。掌握了开放地址法该方法就一目了然了。

set()操作除了存储元素外，还有一个很重要的作用，就是replaceStaleEntry()和cleanSomeSlots()，这两个方法可以清除掉key == null 的实例，防止内存泄漏。在set()方法中还有一个变量很重要：threadLocalHashCode，定义如下：

```
private final int threadLocalHashCode = nextHashCode();

```

从名字上面我们可以看出threadLocalHashCode应该是ThreadLocal的散列值，定义为final，表示ThreadLocal一旦创建其散列值就已经确定了，生成过程则是调用nextHashCode()：

```
    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

```

nextHashCode表示分配下一个ThreadLocal实例的threadLocalHashCode的值，HASH_INCREMENT则表示分配两个ThradLocal实例的threadLocalHashCode的增量，从nextHashCode就可以看出他们的定义。

getEntry()

```
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

```

由于采用了开放定址法，所以当前key的散列值和元素在数组的索引并不是完全对应的，首先取一个探测数（key的散列值），如果所对应的key就是我们所要找的元素，则返回，否则调用getEntryAfterMiss()，如下：

```
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

```

这里有一个重要的地方，当key == null时，调用了expungeStaleEntry()方法，该方法用于处理key == null，有利于GC回收，能够有效地避免内存泄漏。

### get()

> 返回当前线程所对应的线程变量

```
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();

        // 获取当前线程的成员变量 threadLocal
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 从当前线程的ThreadLocalMap获取相对应的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")

                // 获取目标值        
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

```

首先通过当前线程获取所对应的成员变量ThreadLocalMap，然后通过ThreadLocalMap获取当前ThreadLocal的Entry，最后通过所获取的Entry获取目标值result。

getMap()方法可以获取当前线程所对应的ThreadLocalMap，如下：

```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

```

### set(T value)

> 设置当前线程的线程局部变量的值。

```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

```

获取当前线程所对应的ThreadLocalMap，如果不为空，则调用ThreadLocalMap的set()方法，key就是当前ThreadLocal，如果不存在，则调用createMap()方法新建一个，如下：

```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

```

### initialValue()

> 返回该线程局部变量的初始值。

```
    protected T initialValue() {
        return null;
    }

```

该方法定义为protected级别且返回为null，很明显是要子类实现它的，所以我们在使用ThreadLocal的时候一般都应该覆盖该方法。该方法不能显示调用，只有在第一次调用get()或者set()方法时才会被执行，并且仅执行1次。

### remove()

> 将当前线程局部变量的值删除。

```
    public void remove() {
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)
            m.remove(this);
    }

```

该方法的目的是减少内存的占用。当然，我们不需要显示调用该方法，因为一个线程结束后，它所对应的局部变量就会被垃圾回收。

## ThreadLocal为什么会内存泄漏

前面提到每个Thread都有一个ThreadLocal.ThreadLocalMap的map，该map的key为ThreadLocal实例，它为一个弱引用，我们知道弱引用有利于GC回收。当ThreadLocal的key == null时，GC就会回收这部分空间，但是value却不一定能够被回收，因为他还与Current Thread存在一个强引用关系，如下（图片来自[http://www.jianshu.com/p/ee8c9dccc953](http://www.jianshu.com/p/ee8c9dccc953)）：

![](https://img-blog.csdn.net/20171005154856389?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由于存在这个强引用关系，会导致value无法回收。如果这个线程对象不会销毁那么这个强引用关系则会一直存在，就会出现内存泄漏情况。所以说只要这个线程对象能够及时被GC回收，就不会出现内存泄漏。如果碰到线程池，那就更坑了。

那么要怎么避免这个问题呢？

在前面提过，在ThreadLocalMap中的setEntry()、getEntry()，如果遇到key == null的情况，会对value设置为null。当然我们也可以显示调用ThreadLocal的remove()方法进行处理。

下面再对ThreadLocal进行简单的总结：

> *   ThreadLocal 不是用于解决共享变量的问题的，也不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制。这点至关重要。
> *   每个Thread内部都有一个ThreadLocal.ThreadLocalMap类型的成员变量，该成员变量用来存储实际的ThreadLocal变量副本。
> *   ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要木得视为每一个线程隔离一个类的实例，这个实例的作用范围仅限于线程内部。

有关于JMM内存模型的详细介绍将在下一章讲述。

本文_转载自_[并发编程网 – ifeve.com](http://ifeve.com/)


