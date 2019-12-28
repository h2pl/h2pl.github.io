---
title: 后端技术杂谈4：Elasticsearch与solr入门实践
date: 2019-10-13 15:04:26 # 文章生成时间，一般不改
categories:
    - 后端技术
tags:
    - 大后端
---
# 阮一峰：全文搜索引擎 Elasticsearch 入门教程



阅读 1093

收藏 76

2017-08-23



原文链接：[www.ruanyifeng.com](https://link.juejin.im/?target=http%3A%2F%2Fwww.ruanyifeng.com%2Fblog%2F2017%2F08%2Felasticsearch.html)

[9月7日-8日 北京，与 Google Twitch 等团队技术大咖面对面www.bagevent.com](https://www.bagevent.com/event/1291755?bag_track=juejin)



[全文搜索](https://link.juejin.im/?target=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E5%2585%25A8%25E6%2596%2587%25E6%2590%259C%25E7%25B4%25A2%25E5%25BC%2595%25E6%2593%258E)属于最常见的需求，开源的 [Elasticsearch](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2F) （以下简称 Elastic）是目前全文搜索引擎的首选。

它可以快速地储存、搜索和分析海量数据。维基百科、Stack Overflow、Github 都采用它。

![](https://user-gold-cdn.xitu.io/2017/8/23/1744dd6de08a90846583b11bd4638e8f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Elastic 的底层是开源库 [Lucene](https://link.juejin.im/?target=https%3A%2F%2Flucene.apache.org%2F)。但是，你没法直接用 Lucene，必须自己写代码去调用它的接口。Elastic 是 Lucene 的封装，提供了 REST API 的操作接口，开箱即用。

本文从零开始，讲解如何使用 Elastic 搭建自己的全文搜索引擎。每一步都有详细的说明，大家跟着做就能学会。

## 一、安装

Elastic 需要 Java 8 环境。如果你的机器还没安装 Java，可以参考[这篇文章](https://link.juejin.im/?target=https%3A%2F%2Fwww.digitalocean.com%2Fcommunity%2Ftutorials%2Fhow-to-install-java-with-apt-get-on-debian-8)，注意要保证环境变量`JAVA_HOME`正确设置。

安装完 Java，就可以跟着[官方文档](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2Fcurrent%2Fzip-targz.html)安装 Elastic。直接下载压缩包比较简单。

> ```
>  $ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.1.zip$ unzip elasticsearch-5.5.1.zip$ cd elasticsearch-5.5.1/ 
> ```

接着，进入解压后的目录，运行下面的命令，启动 Elastic。

> ```
>  $ ./bin/elasticsearch
> ```

如果这时[报错](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fspujadas%2Felk-docker%2Fissues%2F92)"max virtual memory areas vm.max_map_count [65530] is too low"，要运行下面的命令。

> ```
>  $ sudo sysctl -w vm.max_map_count=262144
> ```

如果一切正常，Elastic 就会在默认的9200端口运行。这时，打开另一个命令行窗口，请求该端口，会得到说明信息。

> ```
>  $ curl localhost:9200 {  "name" : "atntrTf",  "cluster_name" : "elasticsearch",  "cluster_uuid" : "tf9250XhQ6ee4h7YI11anA",  "version" : {    "number" : "5.5.1",    "build_hash" : "19c13d0",    "build_date" : "2017-07-18T20:44:24.823Z",    "build_snapshot" : false,    "lucene_version" : "6.6.0"  },  "tagline" : "You Know, for Search"}
> ```

上面代码中，请求9200端口，Elastic 返回一个 JSON 对象，包含当前节点、集群、版本等信息。

按下 Ctrl + C，Elastic 就会停止运行。

默认情况下，Elastic 只允许本机访问，如果需要远程访问，可以修改 Elastic 安装目录的`config/elasticsearch.yml`文件，去掉`network.host`的注释，将它的值改成`0.0.0.0`，然后重新启动 Elastic。

> ```
>  network.host: 0.0.0.0
> ```

上面代码中，设成`0.0.0.0`让任何人都可以访问。线上服务不要这样设置，要设成具体的 IP。

## 二、基本概念

### 2.1 Node 与 Cluster

Elastic 本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个 Elastic 实例。

单个 Elastic 实例称为一个节点（node）。一组节点构成一个集群（cluster）。

### 2.2 Index

Elastic 会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。

所以，Elastic 数据管理的顶层单位就叫做 Index（索引）。它是单个数据库的同义词。每个 Index （即数据库）的名字必须是小写。

下面的命令可以查看当前节点的所有 Index。

> ```
>  $ curl -X GET 'http://localhost:9200/_cat/indices?v'
> ```

### 2.3 Document

Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。

Document 使用 JSON 格式表示，下面是一个例子。

> ```
>  {  "user": "张三",  "title": "工程师",  "desc": "数据库管理"}
> ```

同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

### 2.4 Type

Document 可以分组，比如`weather`这个 Index 里面，可以按城市分组（北京和上海），也可以按气候分组（晴天和雨天）。这种分组就叫做 Type，它是虚拟的逻辑分组，用来过滤 Document。

不同的 Type 应该有相似的结构（schema），举例来说，`id`字段不能在这个组是字符串，在另一个组是数值。这是与关系型数据库的表的[一个区别](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Fguide%2Fcurrent%2Fmapping.html)。性质完全不同的数据（比如`products`和`logs`）应该存成两个 Index，而不是一个 Index 里面的两个 Type（虽然可以做到）。

下面的命令可以列出每个 Index 所包含的 Type。

> ```
>  $ curl 'localhost:9200/_mapping?pretty=true'
> ```

根据[规划](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fblog%2Findex-type-parent-child-join-now-future-in-elasticsearch)，Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。

## 三、新建和删除 Index

新建 Index，可以直接向 Elastic 服务器发出 PUT 请求。下面的例子是新建一个名叫`weather`的 Index。

> ```
>  $ curl -X PUT 'localhost:9200/weather'
> ```

服务器返回一个 JSON 对象，里面的`acknowledged`字段表示操作成功。

> ```
>  {  "acknowledged":true,  "shards_acknowledged":true}
> ```

然后，我们发出 DELETE 请求，删除这个 Index。

> ```
>  $ curl -X DELETE 'localhost:9200/weather'
> ```

## 四、中文分词设置

首先，安装中文分词插件。这里使用的是 [ik](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fmedcl%2Felasticsearch-analysis-ik%2F)，也可以考虑其他插件（比如 [smartcn](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Fplugins%2Fcurrent%2Fanalysis-smartcn.html)）。

> ```
>  $ ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.1/elasticsearch-analysis-ik-5.5.1.zip
> ```

上面代码安装的是5.5.1版的插件，与 Elastic 5.5.1 配合使用。

接着，重新启动 Elastic，就会自动加载这个新安装的插件。

然后，新建一个 Index，指定需要分词的字段。这一步根据数据结构而异，下面的命令只针对本文。基本上，凡是需要搜索的中文字段，都要单独设置一下。

> ```
>  $ curl -X PUT 'localhost:9200/accounts' -d '{  "mappings": {    "person": {      "properties": {        "user": {          "type": "text",          "analyzer": "ik_max_word",          "search_analyzer": "ik_max_word"        },        "title": {          "type": "text",          "analyzer": "ik_max_word",          "search_analyzer": "ik_max_word"        },        "desc": {          "type": "text",          "analyzer": "ik_max_word",          "search_analyzer": "ik_max_word"        }      }    }  }}'
> ```

上面代码中，首先新建一个名称为`accounts`的 Index，里面有一个名称为`person`的 Type。`person`有三个字段。

> *   user
> *   title
> *   desc

这三个字段都是中文，而且类型都是文本（text），所以需要指定中文分词器，不能使用默认的英文分词器。

Elastic 的分词器称为 [analyzer](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2Fcurrent%2Fanalysis.html)。我们对每个字段指定分词器。

> ```
>  "user": {  "type": "text",  "analyzer": "ik_max_word",  "search_analyzer": "ik_max_word"}
> ```

上面代码中，`analyzer`是字段文本的分词器，`search_analyzer`是搜索词的分词器。`ik_max_word`分词器是插件`ik`提供的，可以对文本进行最大数量的分词。

## 五、数据操作

### 5.1 新增记录

向指定的 /Index/Type 发送 PUT 请求，就可以在 Index 里面新增一条记录。比如，向`/accounts/person`发送请求，就可以新增一条人员记录。

> ```
>  $ curl -X PUT 'localhost:9200/accounts/person/1' -d '{  "user": "张三",  "title": "工程师",  "desc": "数据库管理"}' 
> ```

服务器返回的 JSON 对象，会给出 Index、Type、Id、Version 等信息。

> ```
>  {  "_index":"accounts",  "_type":"person",  "_id":"1",  "_version":1,  "result":"created",  "_shards":{"total":2,"successful":1,"failed":0},  "created":true}
> ```

如果你仔细看，会发现请求路径是`/accounts/person/1`，最后的`1`是该条记录的 Id。它不一定是数字，任意字符串（比如`abc`）都可以。

新增记录的时候，也可以不指定 Id，这时要改成 POST 请求。

> ```
>  $ curl -X POST 'localhost:9200/accounts/person' -d '{  "user": "李四",  "title": "工程师",  "desc": "系统管理"}'
> ```

上面代码中，向`/accounts/person`发出一个 POST 请求，添加一个记录。这时，服务器返回的 JSON 对象里面，`_id`字段就是一个随机字符串。

> ```
>  {  "_index":"accounts",  "_type":"person",  "_id":"AV3qGfrC6jMbsbXb6k1p",  "_version":1,  "result":"created",  "_shards":{"total":2,"successful":1,"failed":0},  "created":true}
> ```

注意，如果没有先创建 Index（这个例子是`accounts`），直接执行上面的命令，Elastic 也不会报错，而是直接生成指定的 Index。所以，打字的时候要小心，不要写错 Index 的名称。

### 5.2 查看记录

向`/Index/Type/Id`发出 GET 请求，就可以查看这条记录。

> ```
>  $ curl 'localhost:9200/accounts/person/1?pretty=true'
> ```

上面代码请求查看`/accounts/person/1`这条记录，URL 的参数`pretty=true`表示以易读的格式返回。

返回的数据中，`found`字段表示查询成功，`_source`字段返回原始记录。

> ```
>  {  "_index" : "accounts",  "_type" : "person",  "_id" : "1",  "_version" : 1,  "found" : true,  "_source" : {    "user" : "张三",    "title" : "工程师",    "desc" : "数据库管理"  }}
> ```

如果 Id 不正确，就查不到数据，`found`字段就是`false`。

> ```
>  $ curl 'localhost:9200/weather/beijing/abc?pretty=true' {  "_index" : "accounts",  "_type" : "person",  "_id" : "abc",  "found" : false}
> ```

### 5.3 删除记录

删除记录就是发出 DELETE 请求。

> ```
>  $ curl -X DELETE 'localhost:9200/accounts/person/1'
> ```

这里先不要删除这条记录，后面还要用到。

### 5.4 更新记录

更新记录就是使用 PUT 请求，重新发送一次数据。

> ```
>  $ curl -X PUT 'localhost:9200/accounts/person/1' -d '{    "user" : "张三",    "title" : "工程师",    "desc" : "数据库管理，软件开发"}'  {  "_index":"accounts",  "_type":"person",  "_id":"1",  "_version":2,  "result":"updated",  "_shards":{"total":2,"successful":1,"failed":0},  "created":false}
> ```

上面代码中，我们将原始数据从"数据库管理"改成"数据库管理，软件开发"。 返回结果里面，有几个字段发生了变化。

> ```
>  "_version" : 2,"result" : "updated","created" : false
> ```

可以看到，记录的 Id 没变，但是版本（version）从`1`变成`2`，操作类型（result）从`created`变成`updated`，`created`字段变成`false`，因为这次不是新建记录。

## 六、数据查询

### 6.1 返回所有记录

使用 GET 方法，直接请求`/Index/Type/_search`，就会返回所有记录。

> ```
>  $ curl 'localhost:9200/accounts/person/_search' {  "took":2,  "timed_out":false,  "_shards":{"total":5,"successful":5,"failed":0},  "hits":{    "total":2,    "max_score":1.0,    "hits":[      {        "_index":"accounts",        "_type":"person",        "_id":"AV3qGfrC6jMbsbXb6k1p",        "_score":1.0,        "_source": {          "user": "李四",          "title": "工程师",          "desc": "系统管理"        }      },      {        "_index":"accounts",        "_type":"person",        "_id":"1",        "_score":1.0,        "_source": {          "user" : "张三",          "title" : "工程师",          "desc" : "数据库管理，软件开发"        }      }    ]  }}
> ```

上面代码中，返回结果的 `took`字段表示该操作的耗时（单位为毫秒），`timed_out`字段表示是否超时，`hits`字段表示命中的记录，里面子字段的含义如下。

> *   `total`：返回记录数，本例是2条。
> *   `max_score`：最高的匹配程度，本例是`1.0`。
> *   `hits`：返回的记录组成的数组。

返回的记录中，每条记录都有一个`_score`字段，表示匹配的程序，默认是按照这个字段降序排列。

### 6.2 全文搜索

Elastic 的查询非常特别，使用自己的[查询语法](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2F5.5%2Fquery-dsl.html)，要求 GET 请求带有数据体。

> ```
>  $ curl 'localhost:9200/accounts/person/_search'  -d '{  "query" : { "match" : { "desc" : "软件" }}}'
> ```

上面代码使用 [Match 查询](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2F5.5%2Fquery-dsl-match-query.html)，指定的匹配条件是`desc`字段里面包含"软件"这个词。返回结果如下。

> ```
>  {  "took":3,  "timed_out":false,  "_shards":{"total":5,"successful":5,"failed":0},  "hits":{    "total":1,    "max_score":0.28582606,    "hits":[      {        "_index":"accounts",        "_type":"person",        "_id":"1",        "_score":0.28582606,        "_source": {          "user" : "张三",          "title" : "工程师",          "desc" : "数据库管理，软件开发"        }      }    ]  }}
> ```

Elastic 默认一次返回10条结果，可以通过`size`字段改变这个设置。

> ```
>  $ curl 'localhost:9200/accounts/person/_search'  -d '{  "query" : { "match" : { "desc" : "管理" }},  "size": 1}'
> ```

上面代码指定，每次只返回一条结果。

还可以通过`from`字段，指定位移。

> ```
>  $ curl 'localhost:9200/accounts/person/_search'  -d '{  "query" : { "match" : { "desc" : "管理" }},  "from": 1,  "size": 1}'
> ```

上面代码指定，从位置1开始（默认是从位置0开始），只返回一条结果。

### 6.3 逻辑运算

如果有多个搜索关键字， Elastic 认为它们是`or`关系。

> ```
>  $ curl 'localhost:9200/accounts/person/_search'  -d '{  "query" : { "match" : { "desc" : "软件 系统" }}}'
> ```

上面代码搜索的是`软件 or 系统`。

如果要执行多个关键词的`and`搜索，必须使用[布尔查询](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2F5.5%2Fquery-dsl-bool-query.html)。

> ```
>  $ curl 'localhost:9200/accounts/person/_search'  -d '{  "query": {    "bool": {      "must": [        { "match": { "desc": "软件" } },        { "match": { "desc": "系统" } }      ]    }  }}'
> ```

## 七、参考链接

*   [ElasticSearch 官方手册](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2Fcurrent%2Fgetting-started.html)
*   [A Practical Introduction to Elasticsearch](https://link.juejin.im/?target=https%3A%2F%2Fwww.elastic.co%2Fblog%2Fa-practical-introduction-to-elasticsearch)

（完）









![](https://upload-images.jianshu.io/upload_images/19687-3a3865f474573947.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)





#### 一、前言

在开发网站/App项目的时候，通常需要搭建搜索服务。比如，新闻类应用需要检索标题/内容，社区类应用需要检索用户/帖子。

对于简单的需求，可以使用数据库的 LIKE 模糊搜索，示例：

> SELECT * FROM news WHERE title LIKE '%法拉利跑车%'

可以查询到所有标题含有 "法拉利跑车" 关键词的新闻，但是这种方式有明显的弊端：

> 1、模糊查询性能极低，当数据量庞大的时候，往往会使数据库服务中断；
> 
> 2、无法查询相关的数据，只能严格在标题中匹配关键词。

因此，需要搭建专门提供搜索功能的服务，具备分词、全文检索等高级功能。 Solr 就是这样一款搜索引擎，可以让你快速搭建适用于自己业务的搜索服务。

#### 二、安装

到官网 [http://lucene.apache.org/solr/](https://link.jianshu.com/?t=http://lucene.apache.org/solr/) 下载安装包，解压并进入 Solr 目录：

> wget 'http://apache.website-solution.net/lucene/solr/6.2.0/solr-6.2.0.tgz'
> 
> tar xvf solr-6.2.0.tgz
> 
> cd solr-6.2.0

目录结构如下：





![](https://upload-images.jianshu.io/upload_images/19687-ddbb880dd1a7bcb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



Solr 6.2 目录结构



启动 Solr 服务之前，确认已经安装 Java 1.8 ：





![](https://upload-images.jianshu.io/upload_images/19687-049501dade838caf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



查看 Java 版本



启动 Solr 服务：

> ./bin/solr start -m 1g

Solr 将默认监听 8983 端口，其中 -m 1g 指定分配给 JVM 的内存为 1 G。

在浏览器中访问 Solr 管理后台：

> http://127.0.0.1:8983/solr/#/





![](https://upload-images.jianshu.io/upload_images/19687-19bdf6ec1077db99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



Solr 管理后台



创建 Solr 应用：

> ./bin/solr create -c my_news

可以在 solr-6.2.0/server/solr 目录下生成 my_news 文件夹，结构如下：





![](https://upload-images.jianshu.io/upload_images/19687-9911b7416917ca06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



my_news 目录结构



同时，可以在管理后台看到 my_news：





![](https://upload-images.jianshu.io/upload_images/19687-81af0fb0b5d89edd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



管理后台



#### 三、创建索引

我们将从 MySQL 数据库中导入数据到 Solr 并建立索引。

首先，需要了解 Solr 中的两个概念： 字段(field) 和 字段类型(fieldType)，配置示例如下：





![](https://upload-images.jianshu.io/upload_images/19687-cbc2ba3d84087319.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



schema.xml 示例



field 指定一个字段的名称、是否索引/存储和字段类型。

fieldType 指定一个字段类型的名称以及在查询/索引的时候可能用到的分词插件。

将 solr-6.2.0\server\solr\my_news\conf 目录下默认的配置文件 managed-schema 重命名为 schema.xml 并加入新的 fieldType：





![](https://upload-images.jianshu.io/upload_images/19687-2657cfb3507d1bae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



分词类型



在 my_news 目录下创建 lib 目录，将用到的分词插件 ik-analyzer-solr5-5.x.jar 加到 lib 目录，结构如下：





![](https://upload-images.jianshu.io/upload_images/19687-3a3e436e33fa9311.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



my_news 目录结构



在 Solr 安装目录下重启服务：

> ./bin/solr restart

可以在管理后台看到新加的类型：





![](https://upload-images.jianshu.io/upload_images/19687-5609a84930ed96f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



text_ik 类型



接下来创建和我们数据库字段对应的 field：title 和 content，类型选为 text_ik：





![](https://upload-images.jianshu.io/upload_images/19687-a46bba01779c0701.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



新建字段 title



将要导入数据的 MySQL 数据库表结构：





![](https://upload-images.jianshu.io/upload_images/19687-ab4dec5179c0f5c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)





编辑 conf/solrconfig.xml 文件，加入类库和数据库配置：





![](https://upload-images.jianshu.io/upload_images/19687-e3dc609b92f395a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



类库







![](https://upload-images.jianshu.io/upload_images/19687-7a145baf9aa36599.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



dataimport config



同时新建数据库连接配置文件 conf/db-mysql-config.xml ，内容如下：





![](https://upload-images.jianshu.io/upload_images/19687-edc3bb352c36e8c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



数据库配置文件



将数据库连接组件 mysql-connector-java-5.1.39-bin.jar 放到 lib 目录下，重启 Solr，访问管理后台，执行全量导入数据： 





![](https://upload-images.jianshu.io/upload_images/19687-a4462a20df0716a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



全量导入数据



创建定时更新脚本：





![](https://upload-images.jianshu.io/upload_images/19687-f437d561069eedd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



定时更新脚本



加入到定时任务，每5分钟增量更新一次索引：





![](https://upload-images.jianshu.io/upload_images/19687-73d93e996f0a132c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



定时任务



在 Solr 管理后台测试搜索结果：





![](https://upload-images.jianshu.io/upload_images/19687-9f003409af70ae7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



分词搜索结果



至此，基本的搜索引擎搭建完毕，外部应用只需通过 http 协议提供查询参数，就可以获取搜索结果。

#### 四、搜索干预

通常需要对搜索结果进行人工干预，比如编辑推荐、竞价排名或者屏蔽搜索结果。Solr 已经内置了 QueryElevationComponent 插件，可以从配置文件中获取搜索关键词对应的干预列表，并将干预结果排在搜索结果的前面。

在 solrconfig.xml 文件中，可以看到：





![](https://upload-images.jianshu.io/upload_images/19687-09494ec9437338cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



干预其请求配置



定义了搜索组件 elevator，应用在 /elevate 的搜索请求中，干预结果的配置文件在 solrconfig.xml 同目录下的 elevate.xml 中，干预配置示例：





![](https://upload-images.jianshu.io/upload_images/19687-3f2587b4bb0dcee3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)





重启 Solr ，当搜索 "关键词" 的时候，id 为 1和 4 的文档将出现在前面，同时 id = 3 的文档被排除在结果之外，可以看到，没有干预的时候，搜索结果为：





![](https://upload-images.jianshu.io/upload_images/19687-b12a6ec2234beaef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



无干预结果



当有搜索干预的时候：





![](https://upload-images.jianshu.io/upload_images/19687-f57a54656abc2f62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



干预结果



通过配置文件干预搜索结果，虽然简单，但是每次更新都要重启 Solr 才能生效，稍显麻烦，我们可以仿照 QueryElevationComponent 类，开发自己的干预组件，例如:从 Redis 中读取干预配置。

#### 五、中文分词

中文的搜索质量，和分词的效果息息相关，可以在 Solr 管理后台测试分词：





![](https://upload-images.jianshu.io/upload_images/19687-bc4dfa9a4801846f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



分词结果测试



上例可以看到，使用 [IKAnalyzer](https://blog.csdn.net/a724888/article/details/80993677) 分词插件，对 “北京科技大学” 分词的测试结果。当用户搜索 “北京”、“科技大学”、“科技大”、“科技”、“大学” 这些关键词的时候，都会搜索到文本内容含 “北京科技大学” 的文档。

常用的中文分词插件有 IKAnalyzer、mmseg4j和 Solr 自带的 smartcn 等，分词效果各有优劣，具体选择哪个，可以根据自己的业务场景，分别测试效果再选择。

分词插件一般都有自己的默认词库和扩展词库，默认词库包含了绝大多数常用的中文词语。如果默认词库无法满足你的需求，比如某些专业领域的词汇，可以在扩展词库中手动添加，这样分词插件就能识别新词语了。





![](https://upload-images.jianshu.io/upload_images/19687-ac9e935a3b98661c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



分词插件扩展词库配置示例



分词插件还可以指定停止词库，将某些无意义的词汇剔出分词结果，比如：“的”、“哼” 等，例如：





![](https://upload-images.jianshu.io/upload_images/19687-34e025db9e4db451.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



去除无意义的词



#### 六、总结

以上介绍了 Solr 最常用的一些功能，Solr 本身还有很多其他丰富的功能，比如分布式部署。

希望对你有所帮助。

#### 七、附录

1、参考资料：

[https://wiki.apache.org/solr/](https://link.jianshu.com/?t=https://wiki.apache.org/solr/)

[http://lucene.apache.org/solr/quickstart.html](https://link.jianshu.com/?t=http://lucene.apache.org/solr/quickstart.html)

[https://cwiki.apache.org/confluence/display/solr/Apache+Solr+Reference+Guide](https://link.jianshu.com/?t=https://cwiki.apache.org/confluence/display/solr/Apache+Solr+Reference+Guide)

2、上述 Demo 中用到的所有配置文件、Jar 包：

[https://github.com/Ceelog/OpenSchool/blob/master/my_news.zip](https://link.jianshu.com/?t=https://github.com/Ceelog/OpenSchool/blob/master/my_news.zip)

3、还有疑问？联系作者微博/微信 [@Ceelog](https://link.jianshu.com/?t=http://weibo.com/ceelog/)




## 微信公众号

### 个人公众号：程序员黄小斜

微信公众号【程序员黄小斜】新生代青年聚集地，程序员成长充电站。作者黄小斜，职业是阿里程序员，身份是斜杠青年，希望和更多的程序员交朋友，一起进步和成长！这一次，我们一起出发。

关注公众号后回复“2019”领取我这两年整理的学习资料，涵盖自学编程、求职面试、算法刷题、Java技术、计算机基础和考研等8000G资料合集。

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)


### 技术公众号：Java技术江湖

微信公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，专注于 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

关注公众号后回复“PDF”即可领取200+页的《Java工程师面试指南》强烈推荐，几乎涵盖所有Java工程师必知必会的知识点。

![](https://img-blog.csdnimg.cn/20190805090108984.jpg)

<script src="https://my.openwrite.cn/js/readmore.js" type="text/javascript"></script>
<script>
    const btw = new BTWPlugin();
    btw.init({
        id: 'container',
        blogId: '15310-1577469423472-640',
        name: '程序员黄小斜',
        qrcode: 'https://s2.ax1x.com/2019/12/28/le9CwT.jpg',
        keyword: '验证码',
    });
</script>