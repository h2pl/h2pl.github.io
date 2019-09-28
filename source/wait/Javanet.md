---
title: Java网络编程和NIO详解开篇：Java网络编程基础
date: 2018-05-25 11:53:28
tags:
    - Java网络编程
    - NIO
categories:
	- 后端
	- Java网络编程与NIO
---


本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Java网络编程和NIO https://blog.csdn.net/column/details/21963.html

部分代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->

# 计算机网络编程基础

转自：https://mp.weixin.qq.com/s/XXMz5uAFSsPdg38bth2jAA

我们是幸运的，因为我们拥有网络。网络是一个神奇的东西，它改变了你和我的生活方式，改变了整个世界。 然而，网络的无标度和小世界特性使得它又是复杂的，无所不在，无所不能，以致于我们无法区分甚至无法描述。

对于一个码农而言，了解网络的基础知识可能还是从了解定义开始，认识OSI的七层协议模型，深入Socket内部，进而熟练地进行网络编程。

关于网络

关于网络，在词典中的定义是这样的：

在电的系统中，由若干元件组成的用来使电信号按一定要求传输的电路或这种电路的部分，叫网络。

作为一名从事过TMN开发的通信专业毕业生，固执地认为网络是从通信系统中诞生的。通信是人与人之间通过某种媒介进行的信息交流与传递。传统的通信网络（即电话网络）是由传输、交换和终端三大部分组成，通信网络是指将各个孤立的设备进行物理连接，实现信息交换的链路，从而达到资源共享和通信的目的。通信网络可以从覆盖范围，拓扑结构，交换方式等诸多视角进行分类...... 满满的回忆，还是留在书架上吧。

网络的概念外延被不断的放大着，抽象的思维能力是人们创新乃至创造的根源。网络用来表示诸多对象及其相互联系，数学上的图，物理学上的模型,交通网络，人际网络，城市网络等等，总之，网络被总结成从同类问题中抽象出来用数学中的图论科学来表达并研究的一种模型。

很多伙伴认为，了解这些之后呢，然并卵。我们关心的只是计算机网络，算机网络是用通信线路和设备将分布在不同地点的多台计算机系统互相连接起来，按照网络协议，分享软硬件功能，最终实现资源共享的系统。特别的，我们谈到的网络只是互联网——Internet，或者移动互联网，需要的是写互连网应用程序。但是，一位工作了五六年的编程高手曾对我说，现在终于了解到基础知识有多重要，技术在不断演进，而相对不变的就是那些原理和编程模型了。

老码农深以为然，编程实践就是从具体到抽象，再到具体，循环往复，螺旋式上升的过程。了解前世今生，只是为了可能触摸到“势”。基础越扎实，建筑就会越有想象的空间。 对于网络编程的基础，大概要从OSI的七层协议模型开始了。

## 七层模型

七层模型（OSI，Open System Interconnection参考模型），是参考是国际标准化组织制定的一个用于计算机或通信系统间互联的标准体系。它是一个七层抽象的模型，不仅包括一系列抽象的术语和概念，也包括具体的协议。 经典的描述如下：

简述每一层的含义：

1.  物理层（Physical Layer）：建立、维护、断开物理连接。

2.  数据链路层 (Link)：逻辑连接、进行硬件地址寻址、差错校验等。

3.  网络层 (Network)：进行逻辑寻址，实现不同网络之间的路径选择。

4.  传输层 (Transport)：定义传输数据的协议端口号，及流控和差错校验。

5.  会话层（Session Layer）：建立、管理、终止会话。

6.  表示层（Presentation Layer）：数据的表示、安全、压缩。

7.  应用层 (Application)：网络服务与最终用户的一个接口

每一层利用下一层提供的服务与对等层通信，每一层使用自己的协议。了解了这些，然并卵。但是，这一模型确实是绝大多数网络编程的基础，作为抽象类存在的，而TCP／IP协议栈只是这一模型的一个具体实现。

### TCP/IP 协议模型

TCP/IP是Internet的基础，是一组协议的代名词，包括许多协议，组成了TCP/IP协议栈。TCP／IP 有四层模型和五层模型之说，区别在于数据链路层是否作为独立的一层存在。个人倾向于5层模型，这样2层和3层的交换设备更容易弄明白。当谈到网络的2层或3层交换机的时候，就知道指的是那些协议。

数据是如何传递呢？这就要了解网络层和传输层的协议，我们熟知的IP包结构是这样的： 

![](https://img-blog.csdn.net/20180601221723141?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2E3MjQ4ODg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

IP协议和IP地址是两个不同的概念，这里没有涉及IPV6的。不关注网络安全的话，对这些结构不必耳熟能详的。传输层使用这样的数据包进行传输，传输层又分为面向连接的可靠传输TCP和数据报UDP。TCP的包结构： 

![](https://img-blog.csdn.net/20180601221803983?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2E3MjQ4ODg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

TCP 连接建立的三次握手肯定是必知必会，在系统调优的时候，内核中关于网络的相关参数与这个图息息相关。UDP是一种无连接的传输层协议，提供的是简单不可靠的信息传输。协议结构相对简单，包括源和目标的端口号，长度以及校验和。基于TCP和UDP的数据封装及解析示例如下： 

![](https://img-blog.csdn.net/20180601221848793?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2E3MjQ4ODg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

还是然并卵么？一个数据包的大小了解了，会发现什么呢？PayLoad到底是多少？在设计协议通信的时候，这些都为我们提供了粒度定义的依据。进一步，通过一个例子看看吧。

### 模型解读示例

FTP是一个比较好的例子。为了方便起见，假设两条计算机分别是A 和 B，将使用FTP 将A上的一个文件X传输到B上。

首先，计算机A和B之间要有物理层的连接，可以是有线比如同轴电缆或者双绞线通过RJ-45的电路接口连接，也可以是无线连接例如WIFI。先简化一下，考虑局域网，暂不讨论路由器和交换机以及WIFI热点。这些物理层的连接建立了比特流的原始传输通路。

接下来，数据链路层登场，建立两台计算机的数据链路。如果A和B所在的网络上同时连接着计算机C，D，E等等，A和B之间如何建立的数据链路呢？这一过程就是物理寻址，A要在众多的物理连接中找到B，依赖的是计算机的物理地址即MAC地址，对就是网卡上的MAC地址。以太网采用CSMA/CD方式来传输数据，数据在以太网的局域网中都是以广播方式传输的，整个局域网中的所有节点都会收到该帧，只有目标MAC地址与自己的MAC地址相同的帧才会被接收。A通过差错控制和接入控制找到了B的网卡，建立可靠的数据通路。

那IP地址呢？ 数据链路建立起来了，还需要IP地址么？我们FTP 命令中制定的是IP地址而不是MAC地址呀？IP地址是逻辑地址，包括网络地址和主机地址。如果A和B在不同的局域网中，中间有着多个路由器，A需要对B进行逻辑寻址才可以的。物理地址用于底层的硬件的通信，逻辑地址用于上层的协议间的通信。在以太网中：逻辑地址就是IP地址，物理地址就是MAC 地址。在使用中，两种地址是用一定的算法将他们两个联系起来的。所以，IP是用来在网络上选择路由的，在FTP的命令中，IP中的原地址就是A的IP地址，目标地址就是B的IP地址。这应该就是网络层，负责将分组数据从源端传输到目的端。

A向B传输一个文件时，如果文件中有部分数据丢失，就可能会造成在B上无法正常阅读或使用。所以需要一个可靠的连接，能够确保传输过程的完整性，这就是传输层的TCP协议，FTP就是建立在TCP之上的。TCP的三次握手确定了双方数据包的序号、最大接受数据的大小(window)以及MSS(Maximum Segment Size)。TCP利用IP完成寻址，TCP中的提供了端口号，FTP中目的端口号一般是21。传输层的端口号对应主机进程，指本地主机与远程主机正在进行的会话。

会话层用来建立、维护、管理应用程序之间的会话，主要功能是对话控制和同步，编程中所涉及的session是会话层的具体体现。表示层完成数据的解编码，加解密，压缩解压缩等，例如FTP中bin命令，代表了二进制传输，即所传输层数据的格式。 HTTP协议里body中的Json，XML等都可以认为是表示层。应用层就是具体应用的本身了，FTP中的PUT，GET等命令都是应用的具体功能特性。

简单地，物理层到电缆连接，数据链路层到网卡，网络层路由到主机，传输层到端口，会话层维持会话，表示层表达数据格式，应用层就是具体FTP中的各种命令功能了。

## Socket

了解了7层模型就可以编程了么，拿起编程语言就可以耍了么？刚开始上手尝试还是可以的，如果要进一步，老码农觉得还是看看底层实现的好，因为一切归根到底都会归结为系统调用。到了操作系统层面如何看网络呢？Socket登场了。

在Linux世界，“一切皆文件”，操作系统把网络读写作为IO操作，就像读写文件那样，对外提供出来的编程接口就是Socket。所以，socket（套接字）是通信的基石，是支持TCP/IP协议网络通信的基本操作单元。socket实质上提供了进程通信的端点。进程通信之前，双方首先必须各自创建一个端点，否则是没有办法建立联系并相互通信的。一个完整的socket有一个本地唯一的socket号，这是由操作系统分配的。

从设计模式的角度看， Socket其实是一个外观模式，它把复杂的TCP/IP协议栈隐藏在Socket接口后面，对用户来说，一组简单的Socket接口就是全部。当应用程序创建一个socket时，操作系统就返回一个整数作为描述符（descriptor）来标识这个套接字。然后，应用程序以该描述符为传递参数，通过调用函数来完成某种操作（例如通过网络传送数据或接收输入的数据）。

在许多操作系统中，Socket描述符和其他I/O描述符是集成在一起的，操作系统把socket描述符实现为一个指针数组，这些指针指向内部数据结构。进一步看，操作系统为每个运行的进程维护一张单独的文件描述符表。当进程打开一个文件时，系统把一个指向此文件内部数据结构的指针写入文件描述符表，并把该表的索引值返回给调用者 。

既然Socket和操作系统的IO操作相关，那么各操作系统IO实现上的差异会导致Socket编程上的些许不同。看看我Mac上的Socket.so 会发现和CentOS上的还是些不同的。 

进程进行Socket操作时，也有着多种处理方式，如阻塞式IO，非阻塞式IO，多路复用(select/poll/epoll)，AIO等等。

多路复用往往在提升性能方面有着重要的作用。select系统调用的功能是对多个文件描述符进行监视，当有文件描述符的文件读写操作完成以及发生异常或者超时，该调用会返回这些文件描述符。select 需要遍历所有的文件描述符，就遍历操作而言，复杂度是 O(N)。

epoll相关系统调用是在Linux 2.5 后的某个版本开始引入的。该系统调用针对传统的select/poll不足，设计上作了很大的改动。select/poll 的缺点在于: 

1.  每次调用时要重复地从用户模式读入参数，并重复地扫描文件描述符。

2.  每次在调用开始时，要把当前进程放入各个文件描述符的等待队列。在调用结束后，又把进程从各个等待队列中删除。

epoll 是把 select/poll 单个的操作拆分为 1 个 epollcreate，多个 epollctrl和一个 wait。此外，操作系统内核针对 epoll 操作添加了一个文件系统，每一个或者多个要监视的文件描述符都有一个对应的inode 节点，主要信息保存在 eventpoll 结构中。而被监视的文件的重要信息则保存在 epitem 结构中，是一对多的关系。由于在执行 epollcreate 和 epollctrl 时，已经把用户模式的信息保存到内核了， 所以之后即便反复地调用 epoll_wait，也不会重复地拷贝参数，不会重复扫描文件描述符，也不反复地把当前进程放入/拿出等待队列。

所以，当前主流的Server侧Socket实现大都采用了epoll的方式，例如Nginx， 在配置文件可以显式地看到 `use epoll`。

## 网络编程

了解了7层协议模型和操作系统层面的Socket实现，可以方便我们理解网络编程。

在系统架构的时候，有重要的一环就是拓扑架构，这里涉及了网络等基础设施，那么7层协议下四层就会有助于我们对业务系统网络结构的观察和判断。在系统设计的时候，往往采用面向接口的设计，而接口也往往是基于HTTP协议的Restful API。 那接口的粒度就可以将data segment作为一个约束了，同时可以关注到移动互联网中的弱网环境。

不同的编程语言，有着不同的框架和库，真正的编写网络程序代码并不复杂，例如，用Erlang 中 gen_tcp 用于编写一个简单的Echo服务器： 

```
Start_echo_server()->
         {ok,Listen}= gen_tcp:listen(1234,[binary,{packet,4},{reuseaddr,true},{active,true}]),
         {ok,socket}=get_tcp:accept(Listen),
         gen_tcp:close(Listen),
         loop(Socket).

loop(Socket) ->
         receive
                  {tcp,Socket,Bin} ->
                            io:format(“serverreceived binary = ~p~n”,[Bin])
                            Str= binary_to_term(Bin),
                            io:format(“server  (unpacked) ~p~n”,[Str]),
                            Reply= lib_misc:string2value(Str),
                            io:format(“serverreplying = ~p~n”,[Reply]),
                            gen_tcp:send(Socket,term_to_binary(Reply)),
                            loop(Socket);
                   {tcp_closed,Socket} ->
                            Io:format(“ServerSocket closed ~n”)
         end.
```

然而，写出漂亮的服务器程序仍然是一件非常吃功夫的事情，例如，个人非常喜欢的python Tornado 代码, 在ioloop.py 中有对多路复用的选择：

```
@classmethod
def configurable_default(cls):
        if hasattr(select, "epoll"):
            from tornado.platform.epoll import EPollIOLoop
            return EPollIOLoop
        if hasattr(select, "kqueue"):
            # Python 2.6+ on BSD or Mac
            from tornado.platform.kqueue import KQueueIOLoop
            return KQueueIOLoop
        from tornado.platform.select import SelectIOLoop
        return SelectIOLoop
```

在HTTPServer.py 中同样继承了TCPServer，进而实现了HTTP协议，代码片段如下：

```
class HTTPServer(TCPServer, Configurable,
                httputil.HTTPServerConnectionDelegate):
                ...
    def initialize(self, request_callback, no_keep_alive=False, io_loop=None,
                   xheaders=False, ssl_options=None, protocol=None,
                   decompress_request=False,
                   chunk_size=None, max_header_size=None,
                   idle_connection_timeout=None, body_timeout=None,
                   max_body_size=None, max_buffer_size=None):
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.xheaders = xheaders
        self.protocol = protocol
        self.conn_params = HTTP1ConnectionParameters(
            decompress=decompress_request,
            chunk_size=chunk_size,
            max_header_size=max_header_size,
            header_timeout=idle_connection_timeout or 3600,
            max_body_size=max_body_size,
            body_timeout=body_timeout)
        TCPServer.__init__(self, io_loop=io_loop, ssl_options=ssl_options,
                           max_buffer_size=max_buffer_size,
                           read_chunk_size=chunk_size)
        self._connections = set()
        ...
```

  

# Java网络编程基础

Java网络编程基础
转自并发编程网https://ifeve.com/

Java 网络教程: 基础

Java提供了非常易用的网络API，调用这些API我们可以很方便的通过建立TCP/IP或UDP套接字，在网络之间进行相互通信，其中TCP要比UDP更加常用，但在本教程中我们对这两种方式都有说明。


尽管Java网络API允许我们通过套接字（Socket）打开或关闭网络连接，但所有的网络通信均是基于Java IO类 InputStream和OutputStream实现的。

此外，我们还可以使用Java NIO API中相关的网络类，用法与Java网络API基本类似，Java NIO API可以以非阻塞模式工作，在某些特定的场景中使用非阻塞模式可以获得较大的性能提升。

Java TCP网络基础

通常情况下，客户端打开一个连接到服务器端的TCP/IP连接，然后客户端开始与服务器之间通信，当通信结束后客户端关闭连接，过程如下图所示：

ClientServerOpen ConnectionSend RequestReceive ResponseClose Connection

​客户端通过一个已打开的连接可以发送不止一个请求。事实上在服务器处于接收状态下，客户端可以发送尽可能多的数据，服务器也可以主动关闭连接。

Java中Socket类和ServerSocket类

当客户端想要打开一个连接到服务器的TCP/IP连接时，就要使用到Java Socket类。socket类只需要被告知连接的IP地址和TCP端口，其余的都有Java实现。

假如我们想要打开一个监听服务，来监听客户端连接某些指定TCP端口的连接，那就需要使用Java ServerSocket类。当客户端通过Socket连接服务器端的ServerSocket监听时，服务器端会指定这个连接的一个Socket，此时客户端与服务器端间的通信就变成Socket与Socket之间的通信。

关于Socket类和ServerSocket类会在后面的文章中有详细的介绍。

Java UDP网络基础

UDP的工作方式与TCP相比略有不同。使用UDP通信时，在客户端与服务器之间并没有建立连接的概念，客户端发送到服务器的数据，服务器可能（也可能并没有）收到这些数据，而且客户端也并不知道这些数据是否被服务器成功接收。当服务器向客户端发送数据时也是如此。

正因为是不可靠的数据传输，UDP相比与TCP来说少了很多的协议开销。

在某些场景中，使用无连接的UDP要优于TCP，这些在文章Java UDP DatagramSocket类介绍中会有更多介绍。

当我们想要在Java中使用TCP/IP通过网络连接到服务器时，就需要创建java.net.Socket对象并连接到服务器。假如希望使用Java NIO，也可以创建Java NIO中的SocketChannel对象。

## Socket
创建Socket

下面的示例代码是连接到IP地址为78.64.84.171服务器上的80端口，这台服务器就是我们的Web服务器（www.jenkov.com），而80端口就是Web服务端口。

Socket socket = new Socket("78.46.84.171", 80);
我们也可以像如下示例中使用域名代替IP地址：

Socket socket = new Socket("jenkov.com", 80);

Socket发送数据

要通过Socket发送数据，我们需要获取Socket的输出流（OutputStream），示例代码如下：

Socket socket = new Socket("jenkov.com", 80);
OutputStream out = socket.getOutputStream(); 

out.write("some data".getBytes());
out.flush();
out.close(); 

socket.close();
代码非常简单，但是想要通过网络将数据发送到服务器端，一定不要忘记调用flush()方法。操作系统底层的TCP/IP实现会先将数据放入一个更大的数据缓存块中，而缓存块的大小是与TCP/IP的数据包大小相适应的。（译者注：调用flush()方法只是将数据写入操作系统缓存中，并不保证数据会立即发送）

Socket读取数据

从Socket中读取数据，我们就需要获取Socket的输入流（InputStream），代码如下：

Socket socket = new Socket("jenkov.com", 80);
InputStream in = socket.getInputStream(); 

int data = in.read();
//... read more data... 

in.close();
socket.close();
代码也并不复杂，但需要注意的是，从Socket的输入流中读取数据并不能读取文件那样，一直调用read()方法直到返回-1为止，因为对Socket而言，只有当服务端关闭连接时，Socket的输入流才会返回-1，而是事实上服务器并不会不停地关闭连接。假设我们想要通过一个连接发送多个请求，那么在这种情况下关闭连接就显得非常愚蠢。

因此，从Socket的输入流中读取数据时我们必须要知道需要读取的字节数，这可以通过让服务器在数据中告知发送了多少字节来实现，也可以采用在数据末尾设置特殊字符标记的方式连实现。

关闭Socket

当使用完Socket后我们必须将Socket关闭，断开与服务器之间的连接。关闭Socket只需要调用Socket.close()方法即可，代码如下：

Socket socket = new Socket("jenkov.com", 80); 

socket.close();
Java 网络教程: ServerSocket
用java.net.ServerSocket实现java服务通过TCP/IP监听客户端连接，你也可以用Java NIO 来代替java网络标准API，这时候需要用到 ServerSocketChannel。

创建一个 ServerSocket连接
以下是一个创建ServerSocket类来监听9000端口的一个简单的代码

ServerSocket serverSocket = new ServerSocket(9000);


监听请求的连接
要获取请求的连接需要用ServerSocket.accept()方法。该方法返回一个Socket类，该类具有普通java Socket类的所有特性。代码如下：

ServerSocket serverSocket = new ServerSocket(9000); boolean isStopped = false;while(!isStopped){   Socket clientSocket = serverSocket.accept();    //do something with clientSocket}

对每个调用了accept()方法的类都只获得一个请求的连接。

另外，请求的连接也只能在线程运行的server中调用了accept()方法之后才能够接受请求。线程运行在server中其它所有的方法上的时候都不能接受客户端的连接请求。所以”接受”请求的线程通常都会把Socket的请求连接放入一个工作线程池中，然后再和客户端连接。更多关于多线程服务端设计的文档请参考 java多线程服务

关闭客户端Socket
客户端请求执行完毕，并且不会再有该客户端的其它请求发送过来的时候，就需要关闭Socket连接，这和关闭一个普通的客户端Socket连接一样。如下代码来执行关闭：

socket.close();

关闭服务端Sockets
要关闭服务的时候需要关掉 ServerSocket连接。通过执行如下代码：

serverSocket.close();



## UDP DatagramSocket
DatagramSocket类是java通过UDP通信的途径。UDP仍位于IP层的上面。 你可以用DatagramSocket类发送和接收UDP数据包。

UDP 和TCP

UDP工作方式和TCP有点不同。当你通过TCP发送数据时，你先要创建连接。一旦TCP连接建立了，TCP会保证你的数据传递到对端，否则它将告诉你已发生的错误。

仅仅用UDP来发送数据包（datagrams）到网络间的某个IP地址。你不能保证数据会不会到达。你也不能保证UDP数据包到达接收方的指令。这意味着UDP比TCP有更少的协议开销（无完整检查流）。


当数据传输过程中不在乎数据包是否丢失时，UDP就比较适合这样的数据传输。比如，网上的电视信号的传输。你希望信号到达客户端时尽可能地接近直播。因此，如果丢失一两个画面，你一点都不在乎。你不希望直播延迟，值想确保所有的画面显示在客户端。你宁可跳过丢失的画面，希望一直看到最新的画面。

这种情况也会发生在网上摄像机直播节目中。谁会关心过去发生的什么，你只想显示当前的画面。你不希望比实际情况慢30s结束，只因为你想看到摄像机显示给观众的所有画面。这跟摄像机录像有点不同。从摄像机录制画面到磁盘，你不希望丢失一个画面。你可能还希望有点延迟，如果有重大的情况发生，就不需要倒回去检查画面。

通过DatagramSocket发送数据
通过Java的DatagramSocket类发送数据，首先需要创建DatagramPacket。如下：

1	buffer = new byte[65508];
2	 
3	InetAddress address = new DatagramPacket(buffer, buffer.length, address,9000);
字节缓冲块（字节数组）就是UDP数据包中用来发送的数据。缓冲块上限长度为65508字节，是单一UDP数据包发送的最大的数据量。

数据包构造函数的长度就是缓存块中用于发送的数据的长度。所有多于最大容量的数据都会被忽略。

包含节点（例如服务器）地址的InetAddress实例携带节点（如服务器）的地址发送的UDP数据包。InetAddress类表示一个ip地址（网络地址）。getByName()方法返回带有一个InetAddress实例，该实例带有匹配主机名的ip地址。

端口参数是UDP端口服务器用来接收正在监听的数据。UDP端口和TCP端口是不一样的。一台电脑同时有不同的进程监听UDP和TCP 80端口。

为了发送数据包，你需要创建DatagramSocket来发送数据。如下：

    1	DatagramSocketdatagramSocket = new DatagramSocket();
调用send()方法发送数据，像这样：

    1	datagramSocket.send(packet);
    完整示例：
    
    1	DatagramSocket datagramSocket = new DatagramSocket();
    2	 
    3	byte[] buffer = "0123456789".getBytes();
    4	 
    5	InetAddress receiverAddress = InetAddress.getLocalHost();
    6	 
    7	DataframPacket packet = new DatagramPacket( buffer, buffer.length, receiverAddress,80);
    8	datagramSocket.send(packet);
    
从DatagramSocket获取数据
从DataframSocket获取数据时，首先创建DataframPacket ,然后通过DatagramSocket类的receive()方法接收数据。例如：

    1	DatagramSocket datagramSocket = new DatagramSocket(80);
    2	 
    3	byte[] buffer = new byte[10];
    4	 
    5	DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
    6	 
    7	datagramSocket.receive(packet);
    
注意DatagramSocket是如何通过传递参数80到它的构造器初始化的。这个参数是UDP端口的DatagramSocket用来接收UDP数据包的。像之前提到的，TCP和UDP端口是不一样的，也不重叠。你可以有俩个不同的进程同时在端口80监听TCP和UDP，没有任何冲突。

第二，字节缓存块和DatagramPacket创建了。注意DatagramPacket是没有关于节点如何发送数据的信息的，当创建一个方数据的DatagramPacket时，它会直到这个信息。这就是为什么我们会用DatagramPacket接收数据而不是发送数据。因此没有目标地址是必须的。

最后，调用DatagramSocket的receive()方法。直到数据包接收到为止，这个方法都是阻塞的。

接收的数据位于DatagramPacket的字节缓冲块。缓冲块可以通过调用getData()获得：

1	byte[] buffer = packet.getData();
缓冲块接收了多少的数据需要你去找出来。你用的协议应该定义每个UDP包发多少数据，活着定义一个你能找到的数据结束标记。
一个真正的服务端程序可能会在一个loop中调用receive()方法，传送所有接收到的DatagramPacket到工作的线程池中，就像TCP服务器处理请求连接一样（查看Java Multithreaded Servers获取更多详情）

## URL + URLConnection
HTTP GET和POST
从URLs到本地文件
在java.net包中包含两个有趣的类：URL类和URLConnection类。这两个类可以用来创建客户端到web服务器（HTTP服务器）的连接。下面是一个简单的代码例子：
    
    1	URL url = new URL("http://jenkov.com");
    2	URLConnection urlConnection = url.openConnection();
    3	InputStream input = urlConnection.getInputStream();
    4	int data = input.read();
    5	while(data != -1){
    6	System.out.print((char) data);
    7	data = input.read();
    8	}
    9	input.close();
HTTP GET和POST
默认情况下URLConnection发送一个HTTP GET请求到web服务器。如果你想发送一个HTTP POST请求，要调用URLConnection.setDoOutput(true)方法，如下：

    1	URL url = new URL("http://jenkov.com");
    2	URLConnection urlConnection = url.openConnection();
    3	urlConnection.setDoOutput(true);
一旦你调用了setDoOutput(true)，你就可以打开URLConnection的OutputStream，如下：

    1	OutputStream output = urlConnection.getOutputStream();
你可以使用这个OutputStream向相应的HTTP请求中写任何数据，但你要记得将其转换成URL编码（关于URL编码的解释，自行Google）（译者注：具体名字是：application/x-www-form-urlencoded MIME 格式编码）。
当你写完数据的时候要记得关闭OutputStream。

从URLs到本地文件
URL也被叫做统一资源定位符。如果你的代码不关心文件是来自网络还是来自本地文件系统，URL类是另外一种打开文件的方式。
下面是一个如何使用URL类打开一个本地文件系统文件的例子：

    1	URL url = new URL("file:/c:/data/test.txt");
    2	URLConnection urlConnection = url.openConnection();
    3	InputStream input = urlConnection.getInputStream();
    4	int data = input.read();
    5	while(data != -1){
    6	System.out.print((char) data);
    7	data = input.read();
    8	}
    9	input.close();
注意：这和通过HTTP访问一个web服务器上的文件的唯一不同处就是URL：”file:/c:/data/test.txt”。

## JarURLConnection

Java的JarURLConnection类用来连接Java Jar文件。一旦连接上，你可以获取Jar文件的信息。一个简单的例子如下：

    
    01	String urlString = "http://butterfly.jenkov.com/"
    02	                 + "container/download/"
    03	                 + "jenkov-butterfly-container-2.9.9-beta.jar";
    04	 
    05	URL jarUrl = new URL(urlString);
    06	JarURLConnection connection = new JarURLConnection(jarUrl);
    07	 
    08	Manifest manifest = connection.getManifest();
    09	 
    10	JarFile jarFile = connection.getJarFile();
    11	//do something with Jar file...

## InetAddress

创建一个 InetAddress 实例
InetAddress 的内部方法
InetAddress 是 Java 对 IP 地址的封装。这个类的实例经常和 UDP DatagramSockets 和 Socket，ServerSocket 类一起使用。


创建一个 InetAddress 实例
InetAddress 没有公开的构造方法，因此你必须通过一系列静态方法中的某一个来获取它的实例。
<!–more–>

下面是为一个域名实例化 InetAddres 类的例子：
    
    InetAddress address = InetAddress.getByName("jenkov.com");
当然也会有为匹配某个 IP 地址来实例化一个 InetAddress:

    InetAddress address = InetAddress.getByName("78.46.84.171");
另外，它还有通过获取本地 IP 地址的来获取 InetAddress 的方法（正在运行程序的那台机器）
    
    InetAddress address = InetAddress.getLocalHost();
InetAddress 内部方法
InetAddress 类还拥有大量你可以调用的其它方法。例如：你可以通过调用getAddress()方法来获取 IP 地址的 byte 数组。如果要了解更多的方法，最简单的方式就是读 JavaDoc 文档中关于 InetAddress 类的部分。




