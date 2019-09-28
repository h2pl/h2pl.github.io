---
title: Java网络编程与NIO详解14：深度解读Tomcat中的NIO模型
date: 2018-05-29 11:54:22
tags:
    - Tomcat
    - NIO
categories:
	- 后端
	- Java网络编程与NIO
---
转自：http://www.linkedkeeper.com/detail/blog.action?bid=1046**

本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Java网络编程和NIO https://blog.csdn.net/column/details/21963.html

部分代码会放在我的的Github：https://github.com/h2pl/
://github.com/h2pl/

<!-- more -->


**一、I/O复用模型解读**

Tomcat的NIO是基于I/O复用来实现的。对这点一定要清楚，不然我们的讨论就不在一个逻辑线上。下面这张图学习过I/O模型知识的一般都见过，出自《UNIX网络编程》，I/O模型一共有阻塞式I/O，非阻塞式I/O，I/O复用(select/poll/epoll)，信号驱动式I/O和异步I/O。这篇文章讲的是I/O复用。

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_27fb058a-424a-44d0-8c0f-8ec345f7c446.jpg)

这里先来说下用户态和内核态，直白来讲，如果线程执行的是用户代码，当前线程处在用户态，如果线程执行的是内核里面的代码，当前线程处在内核态。更深层来讲，操作系统为代码所处的特权级别分了4个级别。不过现代操作系统只用到了0和3两个级别。0和3的切换就是用户态和内核态的切换。更详细的可参照《深入理解计算机操作系统》。I/O复用模型，是同步非阻塞，这里的非阻塞是指I/O读写，对应的是recvfrom操作，因为数据报文已经准备好，无需阻塞。说它是同步，是因为，这个执行是在一个线程里面执行的。有时候，还会说它又是阻塞的，实际上是指阻塞在select上面，必须等到读就绪、写就绪等网络事件。有时候我们又说I/O复用是多路复用，这里的多路是指N个连接，每一个连接对应一个channel，或者说多路就是多个channel。复用，是指多个连接复用了一个线程或者少量线程(在Tomcat中是Math.min(2,Runtime.getRuntime().availableProcessors()))。

上面提到的网络事件有连接就绪，接收就绪，读就绪，写就绪四个网络事件。I/O复用主要是通过Selector复用器来实现的，可以结合下面这个图理解上面的叙述。

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f90c676a-08ff-4777-9cb1-d73920977c54.jpg)

**二、TOMCAT对IO模型的支持**

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_b845018f-63ab-4651-ba62-89ceefd82c29.jpg)

tomcat从6以后开始支持NIO模型，实现是基于JDK的java.nio包。这里可以看到对read body 和response body是Blocking的。关于这点在第6.3节源代码阅读有重点介绍。

**三、TOMCAT中NIO的配置与使用**

在Connector节点配置protocol="org.apache.coyote.http11.Http11NioProtocol"，Http11NioProtocol协议下默认最大连接数是10000，也可以重新修改maxConnections的值，同时我们可以设置最大线程数maxThreads，这里设置的最大线程数就是Excutor的线程池的大小。在BIO模式下实际上是没有maxConnections，即使配置也不会生效，BIO模式下的maxConnections是保持跟maxThreads大小一致，因为它是一请求一线程模式。

**四、NioEndpoint组件关系图解读**

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_6ad52ffb-52f7-4c7b-a226-570dc766e2dd.jpg)

我们要理解tomcat的nio最主要就是对NioEndpoint的理解。它一共包含LimitLatch、Acceptor、Poller、SocketProcessor、Excutor5个部分。LimitLatch是连接控制器，它负责维护连接数的计算，nio模式下默认是10000，达到这个阈值后，就会拒绝连接请求。Acceptor负责接收连接，默认是1个线程来执行，将请求的事件注册到事件列表。有Poller来负责轮询，Poller线程数量是cpu的核数Math.min(2,Runtime.getRuntime().availableProcessors())。

由Poller将就绪的事件生成SocketProcessor同时交给Excutor去执行。Excutor线程池的大小就是我们在Connector节点配置的maxThreads的值。

在Excutor的线程中，会完成从socket中读取httprequest，解析成HttpServletRequest对象，分派到相应的servlet并完成逻辑，然后将response通过socket发回client。在从socket中读数据和往socket中写数据的过程，并没有像典型的非阻塞的NIO的那样，注册OP_READ或OP_WRITE事件到主Selector，而是**直接通过socket完成读写，这时是阻塞完成的**，但是在timeout控制上，使用了NIO的Selector机制，但是这个Selector并不是Poller线程维护的主Selector，而是BlockPoller线程中维护的Selector，称之为辅Selector。详细源代码可以参照 第6.3节。

**五、NioEndpoint执行序列图**

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_7f7ff50e-d4db-4213-878c-fc8dc7ab662f.jpg)

在下一小节NioEndpoint源码解读中我们将对步骤1-步骤11依次找到对应的代码来说明。

**六、NioEndpoint源码解读**

6.1、初始化

无论是BIO还是NIO，开始都会初始化连接限制，不可能无限增大，NIO模式下默认是10000。


    
    
    public void startInternal() throws Exception {
        if (!running) {
            //省略代码...
            initializeConnectionLatch();
            //省略代码...
        }
    }
    protected LimitLatch initializeConnectionLatch() {
        if (maxConnections==-1) 
        return null;
        if (connectionLimitLatch==null) {
            connectionLimitLatch = new LimitLatch(getMaxConnections());
        }
        return connectionLimitLatch;
    }





6.2、步骤解读

下面我们着重叙述跟NIO相关的流程，共分为11个步骤，分别对应上面序列图中的步骤。

**步骤1**：绑定IP地址及端口，将ServerSocketChannel设置为阻塞。

这里为什么要设置成阻塞呢，我们一直都在说非阻塞。Tomcat的设计初衷主要是为了操作方便。这样这里就跟BIO模式下一样了。只不过在BIO下这里返回的是Socket，NIO下这里返回的是SocketChannel。





    public void bind() throws Exception {
        //省略代码...
        serverSock.socket().bind(addr,getBacklog());
        serverSock.configureBlocking(true); 
        //省略代码...
        selectorPool.open();
    }




**步骤2**：启动接收线程





    public void startInternal() throws Exception {
        if (!running) {
            //省略代码...
            startAcceptorThreads();
        }
    }
     
    //这个方法实际是在它的超类AbstractEndpoint里面    
    protected final void startAcceptorThreads() {
        int count = getAcceptorThreadCount();
        acceptors = new Acceptor[count];
     
        for (int i = 0; i < count; i++) {
            acceptors[i] = createAcceptor();
            Thread t = new Thread(acceptors[i], getName() + "-Acceptor-" + i);
            t.setPriority(getAcceptorThreadPriority());
            t.setDaemon(getDaemon());
            t.start();
        }
    }   





**步骤3**：ServerSocketChannel.accept()接收新连接





    protected class Acceptor extends AbstractEndpoint.Acceptor {
        @Override
        public void run() {
            while (running) {
                try {
                    //省略代码...
                    SocketChannel socket = null;
                    try {                        
                        socket = serverSock.accept();//接收新连接
                    } catch (IOException ioe) {
                        //省略代码...
                        throw ioe;
                    }
                    //省略代码...
                    if (running && !paused) {
                        if (!setSocketOptions(socket)) {
                            //省略代码...
                        }
                    } else {
                        //省略代码...
                    }
                } catch (SocketTimeoutException sx) {
                } catch (IOException x) {
                    //省略代码...
                } catch (OutOfMemoryError oom) {
                    //省略代码...
                } catch (Throwable t) {
                    //省略代码...
                }
            }
        }
    }





**步骤4**：将接收到的链接通道设置为非阻塞

**步骤5**：构造NioChannel对象

**步骤6**：register注册到轮询线程





    protected boolean setSocketOptions(SocketChannel socket) {
        try {
            socket.configureBlocking(false);//将连接通道设置为非阻塞
            Socket sock = socket.socket();
            socketProperties.setProperties(sock);
            NioChannel channel = nioChannels.poll();//构造NioChannel对象
            //省略代码...
            getPoller0().register(channel);//register注册到轮询线程
        } catch (Throwable t) {
            //省略代码...
        }
        //省略代码...
    }




**步骤7**：构造PollerEvent，并添加到事件队列



    protected ConcurrentLinkedQueue<Runnable> events = new ConcurrentLinkedQueue<Runnable>();
    public void register(final NioChannel socket) {
        //省略代码...
        PollerEvent r = eventCache.poll();
        //省略代码...
        addEvent(r);
    }



**步骤8**：启动轮询线程



    public void startInternal() throws Exception {
        if (!running) {
            //省略代码...
            // Start poller threads
            pollers = new Poller[getPollerThreadCount()];
            for (int i=0; i<pollers.length; i++) {
                pollers[i] = new Poller();
                Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
                pollerThread.setPriority(threadPriority);
                pollerThread.setDaemon(true);
                pollerThread.start();
            }
            //省略代码...
        }
    }   

**步骤9**：取出队列中新增的PollerEvent并注册到Selector



    public static class PollerEvent implements Runnable {
        //省略代码...
        @Override
        public void run() {
            if ( interestOps == OP_REGISTER ) {
                try {
                    socket.getIOChannel().register(socket.getPoller().getSelector(), SelectionKey.OP_READ, key);
                } catch (Exception x) {
                    log.error("", x);
                }
            } else {
                //省略代码...
            }//end if
        }//run
        //省略代码...
    }


**步骤10**：Selector.select()


    public void run() {
        // Loop until destroy() is called
        while (true) {
            try {
                //省略代码...
                try {
                    if ( !close ) {
                        if (wakeupCounter.getAndSet(-1) > 0) {
                            keyCount = selector.selectNow();
                        } else {
                            keyCount = selector.select(selectorTimeout);
                        }
                        //省略代码...
                    }
                    //省略代码...
                } catch ( NullPointerException x ) {
                    //省略代码...
                } catch ( CancelledKeyException x ) {
                    //省略代码...
                } catch (Throwable x) {
                    //省略代码...
                }
                //省略代码...
                Iterator<SelectionKey> iterator =
                            keyCount > 0 ? selector.selectedKeys().iterator() : null;
                         
                while (iterator != null && iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    KeyAttachment attachment = (KeyAttachment)sk.attachment();
                    if (attachment == null) {
                        iterator.remove();
                    } else {
                        attachment.access();
                        iterator.remove();
                        processKey(sk, attachment);//此方法跟下去就是把SocketProcessor交给Excutor去执行
                    }
                }//while
                //省略代码...
            } catch (OutOfMemoryError oom) {
                //省略代码...
            }
        }//while
        //省略代码...
    }





**步骤11**：根据选择的SelectionKey构造SocketProcessor提交到请求处理线程



    public boolean processSocket(NioChannel socket, SocketStatus status, boolean dispatch) {
        try {
            //省略代码...
            SocketProcessor sc = processorCache.poll();
            if ( sc == null ) 
                sc = new SocketProcessor(socket,status);
            else 
                sc.reset(socket,status);
            if ( dispatch && getExecutor()!=null ) 
                getExecutor().execute(sc);
            else 
                sc.run();
        } catch (RejectedExecutionException rx) {
            //省略代码...
        } catch (Throwable t) {
            //省略代码...
        }
        //省略代码...
    }




6.3、NioBlockingSelector和BlockPoller介绍

上面的序列图有个地方我没有描述，就是NioSelectorPool这个内部类，是因为在整体理解tomcat的nio上面在序列图里面不包括它更好理解。在有了上面的基础后，我们在来说下NioSelectorPool这个类，对更深层了解Tomcat的NIO一定要知道它的作用。NioEndpoint对象中维护了一个NioSelecPool对象，这个NioSelectorPool中又维护了一个BlockPoller线程，这个线程就是基于辅Selector进行NIO的逻辑。以执行servlet后，得到response，往socket中写数据为例，最终写的过程调用NioBlockingSelector的write方法。代码如下：

    public int write(ByteBuffer buf, NioChannel socket, long writeTimeout,MutableInteger lastWrite) 
                    throws IOException {  
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());  
        if ( key == null ) throw new IOException("Key no longer registered");  
        KeyAttachment att = (KeyAttachment) key.attachment();  
        int written = 0;  
        boolean timedout = false;  
        int keycount = 1; //assume we can write  
        long time = System.currentTimeMillis(); //start the timeout timer  
        try {  
            while ( (!timedout) && buf.hasRemaining()) {  
                if (keycount > 0) { //only write if we were registered for a write  
                    //直接往socket中写数据  
                    int cnt = socket.write(buf); //write the data  
                    lastWrite.set(cnt);  
                    if (cnt == -1)  
                        throw new EOFException();  
                    written += cnt;  
                    //写数据成功，直接进入下一次循环，继续写  
                    if (cnt > 0) {  
                        time = System.currentTimeMillis(); //reset our timeout timer  
                        continue; //we successfully wrote, try again without a selector  
                    }  
                }  
                //如果写数据返回值cnt等于0，通常是网络不稳定造成的写数据失败  
                try {  
                    //开始一个倒数计数器   
                    if ( att.getWriteLatch()==null || att.getWriteLatch().getCount()==0) 
                        att.startWriteLatch(1);  
                    //将socket注册到辅Selector，这里poller就是BlockSelector线程  
                    poller.add(att,SelectionKey.OP_WRITE);  
                    //阻塞，直至超时时间唤醒，或者在还没有达到超时时间，在BlockSelector中唤醒  
                    att.awaitWriteLatch(writeTimeout,TimeUnit.MILLISECONDS);  
                }catch (InterruptedException ignore) {  
                    Thread.interrupted();  
                }  
                if ( att.getWriteLatch()!=null && att.getWriteLatch().getCount()> 0) {  
                    keycount = 0;  
                }else {  
                    //还没超时就唤醒，说明网络状态恢复，继续下一次循环，完成写socket  
                    keycount = 1;  
                    att.resetWriteLatch();  
                }  
                if (writeTimeout > 0 && (keycount == 0))  
                    timedout = (System.currentTimeMillis() - time) >= writeTimeout;  
            } //while  
            if (timedout)   
                throw new SocketTimeoutException();  
        } finally {  
            poller.remove(att,SelectionKey.OP_WRITE);  
            if (timedout && key != null) {  
                poller.cancelKey(socket, key);  
            }  
        }  
        return written;  
    }





也就是说当socket.write()返回0时，说明网络状态不稳定，这时将socket注册OP_WRITE事件到辅Selector，由BlockPoller线程不断轮询这个辅Selector，直到发现这个socket的写状态恢复了，通过那个倒数计数器，通知Worker线程继续写socket动作。看一下BlockSelector线程的代码逻辑：





    public void run() {  
        while (run) {  
            try {  
                ......  
                Iterator iterator = keyCount > 0 ? selector.selectedKeys().iterator() : null;  
                while (run && iterator != null && iterator.hasNext()) {  
                    SelectionKey sk = (SelectionKey) iterator.next();  
                    KeyAttachment attachment = (KeyAttachment)sk.attachment();  
                    try {  
                        attachment.access();  
                        iterator.remove(); ;  
                        sk.interestOps(sk.interestOps() & (~sk.readyOps()));  
                        if ( sk.isReadable() ) {  
                            countDown(attachment.getReadLatch());  
                        }  
                        //发现socket可写状态恢复，将倒数计数器置位，通知Worker线程继续  
                        if (sk.isWritable()) {  
                            countDown(attachment.getWriteLatch());  
                        }  
                    }catch (CancelledKeyException ckx) {  
                        if (sk!=null) sk.cancel();  
                        countDown(attachment.getReadLatch());  
                        countDown(attachment.getWriteLatch());  
                    }  
                }//while  
            }catch ( Throwable t ) {  
                log.error("",t);  
            }  
        }  
        events.clear();  
        try {  
            selector.selectNow();//cancel all remaining keys  
        }catch( Exception ignore ) {  
            if (log.isDebugEnabled())log.debug("",ignore);  
        }  
    }




使用这个辅Selector主要是减少线程间的切换，同时还可减轻主Selector的负担。

**七、关于性能**

下面这份报告是我们压测的一个结果，跟想象的是不是不太一样？几乎没有差别，实际上NIO优化的是I/O的读写，如果瓶颈不在这里的话，比如传输字节数很小的情况下，BIO和NIO实际上是没有差别的。NIO的优势更在于用少量的线程hold住大量的连接。还有一点，我们在压测的过程中，遇到在NIO模式下刚开始的一小段时间内容，会有错误，这是因为一般的压测工具是基于一种长连接，也就是说比如模拟1000并发，那么同时建立1000个连接，下一时刻再发送请求就是基于先前的这1000个连接来发送，还有TOMCAT的NIO处理是有POLLER线程来接管的，它的线程数一般等于CPU的核数，如果一瞬间有大量并发过来，POLLER也会顿时处理不过来。

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_d89ab30f-92a5-4839-98fa-9d760f1df330.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_6c3edafa-e690-49ec-a872-fab8bb3ae295.jpg)

**八、总结**

NIO只是优化了网络IO的读写，如果系统的瓶颈不在这里，比如每次读取的字节说都是500b，那么BIO和NIO在性能上没有区别。NIO模式是最大化压榨CPU，把时间片都更好利用起来。对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源如内存，有关线程资源可参照这篇文章《一台java服务器可以跑多少个线程》。因此，使用的线程越少越好。而I/O复用模型正是利用少量的线程来管理大量的连接。在对于维护大量长连接的应用里面更适合用基于I/O复用模型NIO，比如web qq这样的应用。所以我们要清楚系统的瓶颈是I/O还是CPU的计算。