---
title: Java网络编程和NIO详解13：浅析Tomcat9请求处理流程与启动部署过程
date: 2018-05-28 11:54:20
tags:
    - Tomcat
categories:
	- 后端
	- Java网络编程与NIO
---

转自：https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes

本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Java网络编程和NIO https://blog.csdn.net/column/details/21963.html

部分代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->

本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Java网络编程和NIO https://blog.csdn.net/column/details/21963.html

相关代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->


《谈谈 Tomcat 架构及启动过程[含部署]》已重新修订!（与本文在 GitHub 同一目录下）包括架构和 Tomcat Start 过程中的 `MapperListener` 相关描述。`Connector` 启动相关的内容与请求处理关系比较紧密，所以就独立出来放在本文中了。

建议结合《谈谈 Tomcat 架构及启动过程[含部署]》一起看!

很多东西在时序图中体现的已经非常清楚了，没有必要再一步一步的作介绍，所以本文以图为主，然后对部分内容加以简单解释。

*   绘制图形使用的工具是 [PlantUML](http://plantuml.com/) + [Visual Studio Code](https://code.visualstudio.com/) + [PlantUML Extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)

本文对 Tomcat 的介绍以 `Tomcat-9.0.0.M22` 为标准。

`Tomcat-9.0.0.M22` 是 Tomcat 目前最新的版本，但尚未发布，它实现了 `Servlet4.0` 及 `JSP2.3` 并提供了很多新特性，需要 1.8 及以上的 JDK 支持等等，详情请查阅 [Tomcat-9.0-doc](https://tomcat.apache.org/tomcat-9.0-doc/index.html)

*   [Overview](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#overview)
*   [Connector Init and Start](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#connector-init-and-start)
*   [Requtst Process](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#requtst-process)
    *   [Acceptor](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#acceptor)
    *   [Poller](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#poller)
    *   [Worker](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#worker)
    *   [Container](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#container)
*   [At last](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#at-last)

* * *

## [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#overview)Overview

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-request-process-model.jpg)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-request-process-model.jpg)

1.  Connector 启动以后会启动一组线程用于不同阶段的请求处理过程。
    1.  `Acceptor` 线程组。用于接受新连接，并将新连接封装一下，选择一个 `Poller` 将新连接添加到 `Poller` 的事件队列中。
    2.  `Poller` 线程组。用于监听 Socket 事件，当 Socket 可读或可写等等时，将 Socket 封装一下添加到 `worker` 线程池的任务队列中。
    3.  `worker` 线程组。用于对请求进行处理，包括分析请求报文并创建 Request 对象，调用容器的 pipeline 进行处理。

*   `Acceptor`、`Poller`、`worker` 所在的 `ThreadPoolExecutor` 都维护在 `NioEndpoint` 中。

## [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#connector-init-and-start)Connector Init and Start

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-connector-start.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-connector-start.png)

1.  `initServerSocket()`，通过 `ServerSocketChannel.open()` 打开一个 ServerSocket，默认绑定到 8080 端口，默认的连接等待队列长度是 100， 当超过 100 个时会拒绝服务。我们可以通过配置 `conf/server.xml` 中 `Connector` 的 `acceptCount` 属性对其进行定制。
2.  `createExecutor()` 用于创建 `Worker` 线程池。默认会启动 10 个 `Worker` 线程，Tomcat 处理请求过程中，Woker 最多不超过 200 个。我们可以通过配置 `conf/server.xml` 中 `Connector` 的 `minSpareThreads` 和 `maxThreads` 对这两个属性进行定制。
3.  `Pollor` 用于检测已就绪的 Socket。 默认最多不超过 2 个，`Math.min(2,Runtime.getRuntime().availableProcessors());`。我们可以通过配置 `pollerThreadCount` 来定制。
4.  `Acceptor` 用于接受新连接。默认是 1 个。我们可以通过配置 `acceptorThreadCount` 对其进行定制。

## [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#requtst-process)Requtst Process

### [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#acceptor)Acceptor

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-request-process-acceptor.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-request-process-acceptor.png)

1.  `Acceptor` 在启动后会阻塞在 `ServerSocketChannel.accept();` 方法处，当有新连接到达时，该方法返回一个 `SocketChannel`。
2.  配置完 Socket 以后将 Socket 封装到 `NioChannel` 中，并注册到 `Poller`,值的一提的是，我们一开始就启动了多个 `Poller` 线程，注册的时候，连接是公平的分配到每个 `Poller` 的。`NioEndpoint` 维护了一个 `Poller` 数组，当一个连接分配给 `pollers[index]` 时，下一个连接就会分配给 `pollers[(index+1)%pollers.length]`.
3.  `addEvent()` 方法会将 Socket 添加到该 `Poller` 的 `PollerEvent` 队列中。到此 `Acceptor` 的任务就完成了。

### [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#poller)Poller

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-request-process-poller.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-request-process-poller.png)

1.  `selector.select(1000)`。当 `Poller` 启动后因为 selector 中并没有已注册的 `Channel`，所以当执行到该方法时只能阻塞。所有的 `Poller` 共用一个 Selector，其实现类是 `sun.nio.ch.EPollSelectorImpl`
2.  `events()` 方法会将通过 `addEvent()` 方法添加到事件队列中的 Socket 注册到 `EPollSelectorImpl`，当 Socket 可读时，`Poller` 才对其进行处理
3.  `createSocketProcessor()` 方法将 Socket 封装到 `SocketProcessor` 中，`SocketProcessor` 实现了 `Runnable` 接口。`worker` 线程通过调用其 `run()` 方法来对 Socket 进行处理。
4.  `execute(SocketProcessor)` 方法将 `SocketProcessor` 提交到线程池，放入线程池的 `workQueue` 中。`workQueue` 是 `BlockingQueue` 的实例。到此 `Poller` 的任务就完成了。

### [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#worker)Worker

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-request-process-worker.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-request-process-worker.png)

1.  `worker` 线程被创建以后就执行 `ThreadPoolExecutor` 的 `runWorker()` 方法，试图从 `workQueue` 中取待处理任务，但是一开始 `workQueue` 是空的，所以 `worker` 线程会阻塞在 `workQueue.take()` 方法。
2.  当新任务添加到 `workQueue`后，`workQueue.take()` 方法会返回一个 `Runnable`，通常是 `SocketProcessor`,然后 `worker` 线程调用 `SocketProcessor` 的 `run()` 方法对 Socket 进行处理。
3.  `createProcessor()` 会创建一个 `Http11Processor`, 它用来解析 Socket，将 Socket 中的内容封装到 `Request` 中。注意这个 `Request` 是临时使用的一个类，它的全类名是 `org.apache.coyote.Request`，
4.  `postParseRequest()` 方法封装一下 Request，并处理一下映射关系(从 URL 映射到相应的 `Host`、`Context`、`Wrapper`)。
    1.  `CoyoteAdapter` 将 Rquest 提交给 `Container` 处理之前，并将 `org.apache.coyote.Request` 封装到 `org.apache.catalina.connector.Request`，传递给 `Container` 处理的 Request 是 `org.apache.catalina.connector.Request`。
    2.  `connector.getService().getMapper().map()`，用来在 `Mapper` 中查询 URL 的映射关系。映射关系会保留到 `org.apache.catalina.connector.Request` 中，`Container` 处理阶段 `request.getHost()` 是使用的就是这个阶段查询到的映射主机，以此类推 `request.getContext()`、`request.getWrapper()` 都是。
5.  `connector.getService().getContainer().getPipeline().getFirst().invoke()` 会将请求传递到 `Container` 处理，当然了 `Container` 处理也是在 `Worker` 线程中执行的，但是这是一个相对独立的模块，所以单独分出来一节。

### [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Tomcat%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.md#container)Container

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-request-process-container.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-request-process-container.png)

1.  需要注意的是，基本上每一个容器的 `StandardPipeline` 上都会有多个已注册的 `Valve`，我们只关注每个容器的 Basic Valve。其他 Valve 都是在 Basic Valve 前执行。
2.  `request.getHost().getPipeline().getFirst().invoke()` 先获取对应的 `StandardHost`，并执行其 pipeline。
3.  `request.getContext().getPipeline().getFirst().invoke()` 先获取对应的 `StandardContext`,并执行其 pipeline。
4.  `request.getWrapper().getPipeline().getFirst().invoke()` 先获取对应的 `StandardWrapper`，并执行其 pipeline。
5.  最值得说的就是 `StandardWrapper` 的 Basic Valve，`StandardWrapperValve`
    1.  `allocate()` 用来加载并初始化 `Servlet`，值的一提的是 Servlet 并不都是单例的，当 Servlet 实现了 `SingleThreadModel` 接口后，`StandardWrapper` 会维护一组 Servlet 实例，这是享元模式。当然了 `SingleThreadModel`在 Servlet 2.4 以后就弃用了。
    2.  `createFilterChain()` 方法会从 `StandardContext` 中获取到所有的过滤器，然后将匹配 Request URL 的所有过滤器挑选出来添加到 `filterChain` 中。
    3.  `doFilter()` 执行过滤链,当所有的过滤器都执行完毕后调用 Servlet 的 `service()` 方法。





# 谈谈 Tomcat 架构及启动过程[含部署]

这个题目命的其实是很大的，写的时候还是很忐忑的，但我尽可能把这个过程描述清楚。因为这是读过源码以后写的总结，在写的过程中可能会忽略一些前提条件，如果有哪些比较突兀就出现，或不好理解的地方可以给我提 Issue，我会尽快补充修订相关内容。

很多东西在时序图中体现的已经非常清楚了，没有必要再一步一步的作介绍，所以本文以图为主，然后对部分内容加以简单解释。

*   绘制图形使用的工具是 [PlantUML](http://plantuml.com/) + [Visual Studio Code](https://code.visualstudio.com/) + [PlantUML Extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)
*   图形 `PlantUML` 源文件：
    1.  [tomcat-architecture.pu](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-architecture.pu)
    2.  [tomcat-init.pu](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-init.pu)
    3.  [tomcat-start.pu](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/start.pu)
    4.  [tomcat-context-start.pu](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-context-start.pu)
    5.  [tomcat-background-thread.pu](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-background-thread.pu)

本文对 Tomcat 的介绍以 `Tomcat-9.0.0.M22` 为标准。

`Tomcat-9.0.0.M22` 是 Tomcat 目前最新的版本，但尚未发布，它实现了 `Servlet4.0` 及 `JSP2.3` 并提供了很多新特性，需要 1.8 及以上的 JDK 支持等等，详情请查阅 [Tomcat-9.0-doc](https://tomcat.apache.org/tomcat-9.0-doc/index.html)

*   [Overview](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#overview)
*   [Tomcat init](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#tomcat-init)
*   [Tomcat Start[Deployment]](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#tomcat-startdeployment)
    *   [Context Start](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#context-start)
*   [Background process](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#background-process)
*   [How to read excellent open source projects](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#how-to-read-excellent-open-source-projects)
*   [At last](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#at-last)
*   [Reference](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#reference)

* * *

## [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#overview)Overview

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-architecture.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-architecture.png)

1.  `Bootstrap` 作为 Tomcat 对外界的启动类,在 `$CATALINA_BASE/bin` 目录下，它通过反射创建 `Catalina` 的实例并对其进行初始化及启动。
2.  `Catalina` 解析 `$CATALINA_BASE/conf/server.xml` 文件并创建 `StandardServer`、`StandardService`、`StandardEngine`、`StandardHost` 等
3.  `StandardServer` 代表的是整个 Servlet 容器，他包含一个或多个 `StandardService`
4.  `StandardService` 包含一个或多个 `Connector`，和一个 `Engine`，`Connector` 和 `Engine` 都是在解析 `conf/server.xml` 文件时创建的，`Engine` 在 Tomcat 的标准实现是 `StandardEngine`
5.  `MapperListener` 实现了 `LifecycleListener` 和 `ContainerListener` 接口用于监听容器事件和生命周期事件。该监听器实例监听所有的容器，包括 `StandardEngine`、`StandardHost`、`StandardContext`、`StandardWrapper`，当容器有变动时，注册容器到 `Mapper`。
6.  `Mapper` 维护了 URL 到容器的映射关系。当请求到来时会根据 `Mapper` 中的映射信息决定将请求映射到哪一个 `Host`、`Context`、`Wrapper`。
7.  `Http11NioProtocol` 用于处理 HTTP/1.1 的请求
8.  `NioEndpoint` 是连接的端点，在请求处理流程中该类是核心类，会重点介绍。
9.  `CoyoteAdapter` 用于将请求从 Connctor 交给 Container 处理。使 Connctor 和 Container 解耦。
10.  `StandardEngine` 代表的是 Servlet 引擎，用于处理 `Connector` 接受的 Request。包含一个或多个 `Host`（虚拟主机）, `Host` 的标准实现是 `StandardHost`。
11.  `StandardHost` 代表的是虚拟主机，用于部署该虚拟主机上的应用程序。通常包含多个 `Context` (Context 在 Tomcat 中代表应用程序)。`Context` 在 Tomcat 中的标准实现是 `StandardContext`。
12.  `StandardContext` 代表一个独立的应用程序，通常包含多个 `Wrapper`，一个 `Wrapper` 容器封装了一个 Servlet，`Wrapper`的标准实现是 `StandardWrapper`。
13.  `StandardPipeline` 组件代表一个流水线，与 `Valve`（阀）结合，用于处理请求。 `StandardPipeline` 中含有多个 `Valve`， 当需要处理请求时，会逐一调用 `Valve` 的 `invoke` 方法对 Request 和 Response 进行处理。特别的，其中有一个特殊的 `Valve` 叫 `basicValve`,每一个标准容器都有一个指定的 `BasicValve`，他们做的是最核心的工作。
    *   `StandardEngine` 的是 `StandardEngineValve`，他用来将 Request 映射到指定的 `Host`;
    *   `StandardHost` 的是 `StandardHostValve`, 他用来将 Request 映射到指定的 `Context`;
    *   `StandardContext` 的是 `StandardContextValve`，它用来将 Request 映射到指定的 `Wrapper`；
    *   `StandardWrapper` 的是 `StandardWrapperValve`，他用来加载 Rquest 所指定的 Servlet,并调用 Servlet 的 `Service` 方法。

## [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#tomcat-init)Tomcat init

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-start.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-start.png)

*   当通过 `./startup.sh` 脚本或直接通过 `java` 命令来启动 `Bootstrap` 时，Tomcat 的启动过程就正式开始了，启动的入口点就是 `Bootstrap` 类的 `main` 方法。
*   启动的过程分为两步，分别是 `init` 和 `start`，本节主要介绍 `init`;

1.  初始化类加载器。[关于 Tomcat 类加载机制，可以参考我之前写的一片文章：[谈谈Java类加载机制](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88Java%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md)]
    1.  通过从 `CatalinaProperties` 类中获取 `common.loader` 等属性，获得类加载器的扫描仓库。`CatalinaProperties` 类在的静态块中调用了 `loadProperties()` 方法，从 `conf/catalina.properties` 文件中加载了属性.(即在类创建的时候属性就已经加载好了)。
    2.  通过 `ClassLoaderFactory` 创建 `URLClassLoader` 的实例
2.  通过反射创建 `Catalina` 的实例并设置 `parentClassLoader`
3.  `setAwait(true)`。设置 `Catalina` 的 `await` 属性为 `true`。在 Start 阶段尾部，若该属性为 `true`，Tomcat 会在 main 线程中监听 `SHUTDOWN` 命令，默认端口是 8005.当收到该命令后执行 `Catalina` 的 `stop()` 方法关闭 Tomcat 服务器。
4.  `createStartDigester()`。`Catalina` 的该方法用于创建一个 Digester 实例，并添加解析 `conf/server.xml` 的 `RuleSet`。Digester 原本是 Apache 的一个开源项目，专门解析 XML 文件的，但我看 Tomcat-9.0.0.M22 中直接将这些类整合到 Tomcat 内部了，而不是引入 jar 文件。Digester 工具的原理不在本文的介绍范围，有兴趣的话可以参考 [The Digester Component - Apache](http://commons.apache.org/proper/commons-digester/index.html) 或 [《How Tomcat works》- Digester [推荐]](https://www.amazon.com/How-Tomcat-Works-Budi-Kurniawan/dp/097521280X) 一章
5.  `parse()` 方法就是 Digester 处理 `conf/server.xml` 创建各个组件的过程。值的一提的是这些组件都是使用反射的方式来创建的。特别的，在创建 Digester 的时候，添加了一些特别的 `rule Set`，用于创建一些十分核心的组件，这些组件在 `conf/server.xml` 中没有但是其作用都比较大，这里做下简单介绍，当 Start 时用到了再详细说明：
    1.  `EngineConfig`。`LifecycleListener` 的实现类,触发 Engine 的生命周期事件后调用，这个监听器没有特别大的作用，就是打印一下日志
    2.  `HostConfig`。`LifecycleListener` 的实现类，触发 Host 的生命周期事件后调用。这个监听器的作用就是部署应用程序，这包括 `conf/<Engine>/<Host>/` 目录下所有的 Context xml 文件 和 `webapps` 目录下的应用程序，不管是 war 文件还是已解压的目录。 另外后台进程对应用程序的热部署也是由该监听器负责的。
    3.  `ContextConfig`。`LifecycleListener` 的实现类，触发 Context 的生命周期事件时调用。这个监听器的作用是配置应用程序，它会读取并合并 `conf/web.xml` 和 应用程序的 `web.xml`，分析 `/WEB-INF/classes/` 和 `/WEB-INF/lib/*.jar`中的 Class 文件的注解，将其中所有的 Servlet、ServletMapping、Filter、FilterMapping、Listener 都配置到 `StandardContext` 中，以备后期使用。当然了 `web.xml` 中还有一些其他的应用程序参数，最后都会一并配置到 `StandardContext` 中。
6.  `reconfigureStartStopExecutor()` 用于重新配置启动和停止子容器的 `Executor`。默认是 1 个线程。我们可以配置 `conf/server.xml` 中 `Engine` 的 `startStopThreads`，来指定用于启动和停止子容器的线程数量，如果配置 0 的话会使用 `Runtime.getRuntime().availableProcessors()` 作为线程数，若配置为负数的话会使用 `Runtime.getRuntime().availableProcessors() + 配置值`，若和小与 1 的话，使用 1 作为线程数。当线程数是 1 时，使用 `InlineExecutorService` 它直接使用当前线程来执行启动停止操作，否则使用 `ThreadPoolExecutor` 来执行，其最大线程数为我们配置的值。
7.  需要注意的是 Host 的 `init` 操作是在 Start 阶段来做的， `StardardHost` 创建好后其 `state` 属性的默认值是 `LifecycleState.NEW`，所以在其调用 `startInternal()` 之前会进行一次初始化。

[](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#tomcat-startdeployment)Tomcat Start[Deployment]

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-start.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-start.png)

1.  图中从 `StandardHost` Start `StandardContext` 的这步其实在真正的执行流程中会直接跳过，因为 `conf/server.xml` 文件中并没有配置任何的 `Context`，所以在 `findChildren()` 查找子容器时会返回空数组，所以之后遍历子容器来启动子容器的 `for` 循环就直接跳过了。
2.  触发 Host 的 `BEFORE_START_EVENT` 生命周期事件，`HostConfig` 调用其 `beforeStart()` 方法创建 `$CATALINA_BASE/webapps`& `$CATALINA_BASE/conf/<Engine>/<Host>/` 目录。
3.  触发 Host 的 `START_EVENT` 生命周期事件，`HostConfig` 调用其 `start()` 方法开始部署已在 `$CATALINA_BASE/webapps` & `$CATALINA_BASE/conf/<Engine>/<Host>/` 目录下的应用程序。
    1.  解析 `$CATALINA_BASE/conf/<Engine>/<Host>/` 目录下所有定义 `Context` 的 XML 文件，并添加到 `StandardHost`。这些 XML 文件称为应用程序描述符。正因为如此，我们可以配置一个虚拟路径来保存应用程序中用到的图片，详细的配置过程请参考 [开发环境配置指南 - 6.3\. 配置图片存放目录](https://github.com/c-rainstorm/OnlineShoppingSystem-Documents/blob/master/%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E4%B8%8E%E6%8A%80%E6%9C%AF%E8%AF%B4%E6%98%8E/%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md#63-%E9%85%8D%E7%BD%AE%E5%9B%BE%E7%89%87%E5%AD%98%E6%94%BE%E7%9B%AE%E5%BD%95)
    2.  部署 `$CATALINA_BASE/webapps` 下所有的 WAR 文件，并添加到 `StandardHost`。
    3.  部署 `$CATALINA_BASE/webapps` 下所有已解压的目录，并添加到 `StandardHost`。
    *   特别的，添加到 `StandardHost` 时，会直接调用 `StandardContext` 的 `start()` 方法来启动应用程序。启动应用程序步骤请看 Context Start 一节。
4.  在 `StandardEngine` 和 `StandardContext` 启动时都会调用各自的 `threadStart()` 方法，该方法会创建一个新的后台线程来处理该该容器和子容器及容器内各组件的后台事件。`StandardEngine` 会直接创建一个后台线程，`StandardContext` 默认是不创建的，和 `StandardEngine` 共用同一个。后台线程处理机制是周期调用组件的 `backgroundProcess()` 方法。详情请看 Background process 一节。
5.  `MapperListener`
    *   `addListeners(engine)` 方法会将该监听器添加到 `StandardEngine` 和它的所有子容器中
    *   `registerHost()` 会注册所有的 `Host` 和他们的子容器到 `Mapper` 中，方便后期请求处理时使用。
    *   当有新的应用(`StandardContext`)添加进来后，会触发 Host 的容器事件，然后通过 `MapperListener` 将新应用的映射注册到 `Mapper` 中。
6.  Start 工作都做完以后 `Catalina` 会创建一个 `CatalinaShutdownHook` 并注册到 JVM。`CatalinaShutdownHook` 继承了 `Thread`,是 `Catalina` 的内部类。其 `run` 方法中直接调用了 `Catalina` 的 `stop()` 方法来关闭整个服务器。注册该 Thread 到 JVM 的原因是防止用户非正常终止 Tomcat，比如直接关闭命令窗口之类的。当直接关闭命令窗口时，操作系统会向 JVM 发送一个终止信号，然后 JVM 在退出前会逐一启动已注册的 ShutdownHook 来关闭相应资源。

### [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#context-start)Context Start

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-context-start.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-context-start.png)

1.  `StandRoot` 类实现了 `WebResourceRoot` 接口，它容纳了一个应用程序的所有资源，通俗的来说就是部署到 `webapps` 目录下对应 `Context` 的目录里的所有资源。因为我对 Tomcat 的资源管理部分暂时不是很感兴趣，所以资源管理相关类只是做了简单了解，并没有深入研究源代码。
2.  `resourceStart()` 方法会对 `StandardRoot` 进行初始配置
3.  `postWorkDirectory()` 用于创建对应的工作目录 `$CATALINA_BASE/work/<Engine>/<Host>/<Context>`, 该目录用于存放临时文件。
4.  `StardardContext` 只是一个容器，而 `ApplicationContext` 则是一个应用程序真正的运行环境，相关类及操作会在请求处理流程看完以后进行补充。
5.  `StardardContext` 触发 `CONFIGURE_START_EVENT` 生命周期事件，`ContextConfig` 开始调用 `configureStart()` 对应用程序进行配置。
    1.  这个过程会解析并合并 `conf/web.xml` & `conf/<Engine>/<Host>/web.xml.default` & `webapps/<Context>/WEB-INF/web.xml` 中的配置。
    2.  配置配置文件中的参数到 `StandardContext`, 其中主要的包括 Servlet、Filter、Listener。
    3.  因为从 Servlet3.0 以后是直接支持注解的，所以服务器必须能够处理加了注解的类。Tomcat 通过分析 `WEB-INF/classes/` 中的 Class 文件和 `WEB-INF/lib/` 下的 jar 包将扫描到的 Servlet、Filter、Listerner 注册到 `StandardContext`。
    4.  `setConfigured(true)`，是非常关键的一个操作，它标识了 Context 的成功配置，若未设置该值为 true 的话，Context 会启动失败。

## [](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/reading-notes/%E8%B0%88%E8%B0%88%20Tomcat%20%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%5B%E5%90%AB%E9%83%A8%E7%BD%B2%5D.md#background-process)Background process

[![](https://github.com/c-rainstorm/blog/raw/tomcat-request-process/res/tomcat-background-thread.png)](https://github.com/c-rainstorm/blog/blob/tomcat-request-process/res/tomcat-background-thread.png)

1.  后台进程的作用就是处理一下 Servlet 引擎中的周期性事件，处理周期默认是 10s。
2.  特别的 `StandardHost` 的 `backgroundProcess()` 方法会触发 Host 的 `PERIODIC_EVENT` 生命周期事件。然后 `HostConfig` 会调用其 `check()` 方法对已加载并进行过重新部署的应用程序进行 `reload` 或对新部署的应用程序进行热部署。热部署跟之前介绍的部署步骤一致， `reload()` 过程只是简单的顺序调用 `setPause(true)、stop()、start()、setPause(false)`，其中 `setPause(true)` 的作用是暂时停止接受请求。



