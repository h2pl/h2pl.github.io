---
title: Spring与SpringMVC源码解析总结
date: 2018-07-08 22:13:58
tags:
	- Spring
categories:
	- 后端
	- 技术总结
---
这篇总结主要是基于我之前Spring和SpringMVC源码系列文章而形成的的。主要是把重要的知识点用自己的话说了一遍，可能会有一些错误，还望见谅和指点。谢谢

更多详细内容可以查看我的专栏文章：#Spring和SpringMVC源码解析
https://blog.csdn.net/column/details/21851.html

<!-- more -->

# Spring和SpringMVC

Spring是一个框架，除了提供IOC和AOP以外，还加入了web等众多内容。

1 IOC：控制反转，改变类实例化的方式，通过xml等配置文件指定接口的实现类，让实现类和代码解耦，通过配置文件灵活调整实现类。

2 AOP: 面向切面编程，将切面代码封装，比如权限验证，日志模块等，这些逻辑重复率大，通过一个增强器封装功能，然后定义需要加入这些功能的切面，切面一般用表达式或者注解去匹配方法，可以完成前置和后置的处理逻辑。

3 SpringMVC是一个web框架，基于Spring之上，实现了web相关的功能，使用dispatcherservlet作为一切请求的处理入口。通过配置viewresolver解析页面，通过配置管理静态文件，还可以注入其他的配置信息，除此之外，springmvc可以访问spring容器的所有bean。


## Spring源码总结

IOC:

1 Spring的bean容器也叫beanfactory，我们常用的applicationcontext实际上内部有一个listablebeanfactory实际存储bean的map。

2 bean加载过程：spring容器加载时先读取配置文件，一般是xml，然后解析xml，找到其中所有bean，依次解析，然后生成每个bean的beandefinition，存在一个map中，根据beanid映射实际bean的map。

3 bean初始化：加载完以后，如果不启用懒加载模式，则默认使用单例加载，在注册完bean以后，可以获取到beandefinition信息，然后根据该信息首先先检查依赖关系，如果依赖其他bean则先加载其他bean，然后通过反射的方式即newinstance创建一个单例bean。

为什么要用反射呢，因为实现类可以通过配置改变，但接口是一致的，使用反射可以避免实现类改变时无法自动进行实例化。

当然，bean也可以使用原型方式加载，使用原型的话，每次创建bean都会是全新的。

AOP:

AOP的切面，切点，增强器一般也是配置在xml文件中的，所以bean容器在解析xml时会找到这些内容，并且首先创建增强器bean的实例。

基于上面创建bean的过程，AOP起到了什么作用呢，或者是是否有参与到其中呢，答案是有的。

在获得beandefinition的时候，spring容器会检查该bean是否有aop切面所修饰，是否有能够匹配切点表达式的方法，如果有的话，在创建bean之前，会将bean重新封装成一个动态代理的对象。

代理类会为bean增加切面中配置的advisor增强器，然后返回bean的时候实际上返回的是一个动态代理对象。

所以我们在调用bean的方法时，会自动织入切面的增强器，当然，动态代理既可以选择jdk增强器，也可以选择cglib增强器。

Spring事务：

spring事务其实是一种特殊的aop方式。在spring配置文件中配置好事务管理器和声明式事务注解后，就可以使用@transactional进行事务方法的处理了。

事务管理器的bean中会配置基本的信息，然后需要配置事务的增强器，不同方法使用不同的增强器。当然如果使用注解的话就不用这么麻烦了。

然后和aop的动态代理方式类似，当Spring容器为bean生成代理时，会注入事务的增强器，其中实际上实现了事务中的begin和commit，所以执行方法的过程实际上就是在事务中进行的。

# SpringMVC源码总结

1 dispatcherservlet概述
SpringMVC使用dispatcherservlet作为唯一如果，在web.xml中进行配置，他继承自frameworkservlet，向上继承自httpservletbean。

httpservletbean为dispatcherservlet加载了来自web.xml配置信息中的信息，保存在servletcontext上下文中，而frameworkservletbean则初始化了spring web的bean容器。

这个容器一般是配置在spring-mvc.xml中的，他独立于spring容器，但是把spring容器作为父容器，所以SpringMVC可以访问spring容器中的各种类。

而dispatcherservlet自己做了什么呢，因为springmvc中配置了很多例如静态文件目录，自动扫描bean注解，以及viewresovler和httpconverter等信息，所以它需要初始化这些策略，如果没有配置则会使用默认值。

2 dispatcherservlet的执行流程

首先web容器会加载指定扫描bean并进行初始化。

当请求进来后，首先执行service方法，然后到dodispatch方法执行请求转发，事实上，spring web容器已经维护了一个map，通过注解@requestmapping映射到对应的bean以及方法上。通过这个map可以获取一个handlerchain，真正要执行的方法被封装成一个handler，并且调用方法前要执行前置的一些过滤器。

最终执行handler方法时实际上就是去执行真正的方法了。

3 viewresolver

解析完请求和执行完方法，会把modelandview对象解析成一个view对象，让后使用view.render方法执行渲染，至于使用什么样的视图解析器，就是由你配置的viewresolver来决定的，一般默认是jspviewresolver。

4 httpmessageconverter

一般配合responsebody使用，可以将数据自动转换为json和xml，根据http请求中适配的数据类型来决定使用哪个转换器。


