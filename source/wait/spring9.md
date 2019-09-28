---
title: Spring源码剖析9：Spring事务源码剖析
date: 2018-06-05 22:29:06
tags:
    - Spring事务
categories:
	- 后端
	- Spring
---

转自：http://www.linkedkeeper.com/detail/blog.action?bid=1045

本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Spring源码解析 https://blog.csdn.net/column/details/21851.html

部分代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->

## 声明式事务使用

Spring事务是我们日常工作中经常使用的一项技术，Spring提供了编程、注解、aop切面三种方式供我们使用Spring事务，其中编程式事务因为对代码入侵较大所以不被推荐使用，注解和aop切面的方式可以基于需求自行选择，我们以注解的方式为例来分析Spring事务的原理和源码实现。

首先我们简单看一下Spring事务的使用方式，配置：


    <tx:annotation-driven transaction-manager="transactionManager"/>
     <bean id="transactionManager" 
             class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
         <property name="dataSource" ref="dataSource"/>
     </bean>





在需要开启事务的方法上加上@Transactional注解即可，这里需要注意的是，当<tx:annotation-driven/>标签在不指定transaction-manager属性的时候，会默认寻找id固定名为transactionManager的bean作为事务管理器，如果没有id为transactionManager的bean并且在使用@Transactional注解时也没有指定value（事务管理器），程序就会报错。当我们在配置两个以上的<tx:annotation-driven/>标签时，如下：



        <tx:annotation-driven transaction-manager="transactionManager1"/>
    <bean id="transactionManager1" 
            class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource1"/>
    </bean>
    <tx:annotation-driven transaction-manager="transactionManager2"/>
    <bean id="transactionManager2" 
            class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource2"/>
    </bean>




这时第一个<tx:annotation-driven/>会生效，也就是当我们使用@Transactional注解时不指定事务管理器，默认使用的事务管理器是transactionManager1，后文分析源码时会具体提到这些注意点。

下面我们开始分析Spring的相关源码，首先看一下对<tx:annotation-driven/>标签的解析，这里需要读者对Spring自定义标签解析的过程有一定的了解，笔者后续也会出相关的文章。锁定TxNamespaceHandler：

## TxNamespaceHandler

(右键可查看大图)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f205abd7-5bc2-4b82-840c-87a96f8351ab.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_9a6ff79d-58b5-4ecd-bf41-ccafde97c44a.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_c1a525fa-56d4-4c85-8208-57558cb53aec.jpg)

## 注册事务功能bean

这个方法比较长，关键的部分做了标记，最外围的if判断限制了<tx:annotation-driven/>标签只能被解析一次，所以只有第一次被解析的标签会生效。蓝色框的部分分别注册了三个BeanDefinition，分别为AnnotationTransactionAttributeSource、TransactionInterceptor和BeanFactoryTransactionAttributeSourceAdvisor，并将前两个BeanDefinition添加到第三个BeanDefinition的属性当中，这三个bean支撑了整个事务功能，后面会详细说明。我们先来看红色框的第个方法：



![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_35702577-8042-4a66-a4eb-337307324810.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_384e595c-2b6d-44cc-bd0b-25fa1f47af5e.jpg)

还记得当<tx:annotation-driven/>标签在不指定transaction-manager属性的时候，会默认寻找id固定名为transactionManager的bean作为事务管理器这个注意事项么，就是在这里实现的。下面我们来看红色框的第二个方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_82a174da-6c9e-4b3c-9a4f-a99228ee6262.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_7288aabb-a34e-4386-a648-de9d9c361743.jpg)

这两个方法的主要目的是注册InfrastructureAdvisorAutoProxyCreator，注册这个类的目的是什么呢？我们看下这个类的层次：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f55221cc-7b52-4479-92eb-3ee52509eb8e.jpg)

## 使用bean的后处理方法获取增强器

我们发现这个类间接实现了BeanPostProcessor接口，我们知道，Spring会保证所有bean在实例化的时候都会调用其postProcessAfterInitialization方法，我们可以使用这个方法包装和改变bean，而真正实现这个方法是在其父类AbstractAutoProxyCreator类中：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_4483fff2-e26d-44eb-8726-64da54fd6e4b.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_a3285183-0bd9-4302-9db9-cd719ac90fdb.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_843d5be7-2cfc-4ca4-ab46-e535d9703705.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_548a4b88-1b9b-43ba-8aad-0a6f0ac1ee2f.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_c6ec857c-f145-49c0-8f50-7b3ac008b907.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_23fadeda-ae10-4108-a358-f1dafe8f1db4.jpg)

上面这个方法相信大家已经看出了它的目的，先找出所有对应Advisor的类的beanName，再通过beanFactory.getBean方法获取这些bean并返回。不知道大家还是否记得在文章开始的时候提到的三个类，其中BeanFactoryTransactionAttributeSourceAdvisor实现了Advisor接口，所以这个bean就会在此被提取出来，而另外两个bean被织入了BeanFactoryTransactionAttributeSourceAdvisor当中，所以也会一起被提取出来，下图为BeanFactoryTransactionAttributeSourceAdvisor类的层次：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f8266cd6-4916-4c93-8b59-6711d6d94ebb.jpg)

## Spring获取匹配的增强器

下面让我们来看Spring如何在所有候选的增强器中获取匹配的增强器：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_ba7d0e1c-474b-442c-893f-251ae510390e.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_0b89068d-863e-46ac-aa39-389a9f0a713e.jpg)

上面的方法中提到引介增强的概念，在此做简要说明，引介增强是一种比较特殊的增强类型，它不是在目标方法周围织入增强，而是为目标类创建新的方法和属性，所以引介增强的连接点是类级别的，而非方法级别的。通过引介增强，我们可以为目标类添加一个接口的实现，即原来目标类未实现某个接口，通过引介增强可以为目标类创建实现该接口的代理，使用方法可以参考文末的引用链接。另外这个方法用两个重载的canApply方法为目标类寻找匹配的增强器，其中第一个canApply方法会调用第二个canApply方法并将第三个参数传为false：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_1961aea7-915a-4abb-bf3f-158a4bc0b79c.jpg)

在上面BeanFactoryTransactionAttributeSourceAdvisor类的层次中我们看到它实现了PointcutAdvisor接口，所以会调用红框中的canApply方法进行判断，第一个参数pca.getPointcut()也就是调用BeanFactoryTransactionAttributeSourceAdvisor的getPointcut方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_dd7d4b92-419d-44c7-b3e5-f2e1897fa473.jpg)

这里的transactionAttributeSource也就是我们在文章开始看到的为BeanFactoryTransactionAttributeSourceAdvisor织入的两个bean中的AnnotationTransactionAttributeSource，我们以TransactionAttributeSourcePointcut作为第一个参数继续跟踪canApply方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f3e359a5-26e5-4a2c-866a-46bac7aad87b.jpg)

我们跟踪pc.getMethodMatcher()方法也就是TransactionAttributeSourcePointcut的getMethodMatcher方法是在它的父类中实现：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_23ae85db-b941-4d08-8aa0-fd0c6980dc1a.jpg)

发现方法直接返回this，也就是下面methodMatcher.matches方法就是调用TransactionAttributeSourcePointcut的matches方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f390cb4e-ca76-49ee-ad9d-d491b5f86ad1.jpg)

在上面我们看到其实这个tas就是AnnotationTransactionAttributeSource，这里的目的其实也就是判断我们的业务方法或者类上是否有@Transactional注解，跟踪AnnotationTransactionAttributeSource的getTransactionAttribute方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_a51a6423-2848-4a9b-bb71-9f7639ef33fa.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_8ec34ac9-3404-4cb9-af16-3bb33b2a02e5.jpg)

方法中的事务声明优先级最高，如果方法上没有声明则在类上寻找：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_c073014d-ff9e-4a88-ab30-f39bf480776f.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_56e3cf18-453f-4845-a03b-de530f22b666.jpg)

this.annotationParsers是在AnnotationTransactionAttributeSource类初始化的时候初始化的：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_a3971a3d-77b9-4689-a6db-afbb70d985a3.jpg)

所以annotationParser.parseTransactionAnnotation就是调用SpringTransactionAnnotationParser的parseTransactionAnnotation方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_d1a502be-3aed-48d6-8321-89e94f942ca3.jpg)

至此，我们终于看到的Transactional注解，下面无疑就是解析注解当中声明的属性了：

## Transactional注解

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_655918f9-40b3-493d-8eca-d5351e005655.jpg)

在这个方法中我们看到了在Transactional注解中声明的各种常用或者不常用的属性的解析，至此，事务的初始化工作算是完成了，下面开始真正的进入执行阶段。

在上文AbstractAutoProxyCreator类的wrapIfNecessary方法中，获取到目标bean匹配的增强器之后，会为bean创建代理，这部分内容我们会在Spring AOP的文章中进行详细说明，在此简要说明方便大家理解，在执行代理类的目标方法时，会调用Advisor的getAdvice获取MethodInterceptor并执行其invoke方法，而我们本文的主角BeanFactoryTransactionAttributeSourceAdvisor的getAdvice方法会返回我们在文章开始看到的为其织入的另外一个bean，也就是TransactionInterceptor，它实现了MethodInterceptor，所以我们分析其invoke方法：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_2fc8d7dd-d830-4214-b8c6-c92c77171a1e.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_60d5b11e-b770-453a-a900-4a5d99619122.jpg)

这个方法很长，但是整体逻辑还是非常清晰的，首选获取事务属性，这里的getTransactionAttrubuteSource()方法的返回值同样是在文章开始我们看到的被织入到TransactionInterceptor中的AnnotationTransactionAttributeSource，在事务准备阶段已经解析过事务属性并保存到缓存中，所以这里会直接从缓存中获取，接下来获取配置的TransactionManager，也就是determineTransactionManager方法，这里如果配置没有指定transaction-manager并且也没有默认id名为transactionManager的bean，就会报错，然后是针对声明式事务和编程式事务的不同处理，创建事务信息，执行目标方法，最后根据执行结果进行回滚或提交操作，我们先分析创建事务的过程。在分析之前希望大家能先去了解一下Spring的事务传播行为，有助于理解下面的源码，这里做一个简要的介绍，更详细的信息请大家自行查阅Spring官方文档，里面有更新详细的介绍。

Spring的事务传播行为定义在Propagation这个枚举类中，一共有七种，分别为：

REQUIRED：业务方法需要在一个容器里运行。如果方法运行时，已经处在一个事务中，那么加入到这个事务，否则自己新建一个新的事务，是默认的事务传播行为。

NOT_SUPPORTED：声明方法不需要事务。如果方法没有关联到一个事务，容器不会为他开启事务，如果方法在一个事务中被调用，该事务会被挂起，调用结束后，原先的事务会恢复执行。

REQUIRESNEW：不管是否存在事务，该方法总汇为自己发起一个新的事务。如果方法已经运行在一个事务中，则原有事务挂起，新的事务被创建。

MANDATORY：该方法只能在一个已经存在的事务中执行，业务方法不能发起自己的事务。如果在没有事务的环境下被调用，容器抛出例外。

SUPPORTS：该方法在某个事务范围内被调用，则方法成为该事务的一部分。如果方法在该事务范围外被调用，该方法就在没有事务的环境下执行。

NEVER：该方法绝对不能在事务范围内执行。如果在就抛例外。只有该方法没有关联到任何事务，才正常执行。

NESTED：如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按REQUIRED属性执行。它使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。内部事务的回滚不会对外部事务造成影响。它只对DataSourceTransactionManager事务管理器起效。

## 开启事务过程

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_708ab424-b604-43f9-bbd5-fb2624db99b9.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_43132c39-82d2-482a-896f-9817c4b95828.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_4d557bcf-70f8-485d-b464-99414abc12a2.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_504e5c5d-bd6b-4213-9f31-85d9373e98ee.jpg)

判断当前线程是否存在事务就是判断记录的数据库连接是否为空并且transactionActive状态为true。

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_a483d06f-ea22-4a76-989e-e5aeee2ec88c.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_4c5b05f6-bfc8-447c-8e3e-d37c56e4fab9.jpg)

REQUIRESNEW会开启一个新事务并挂起原事务，当然开启一个新事务就需要一个新的数据库连接：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_3af47045-58ce-4657-a246-96644fe23ac1.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_850d67a1-6a9f-4eaf-b2b4-e95ec103be74.jpg)

suspend挂起操作主要目的是将当前connectionHolder置为null，保存原有事务信息，以便于后续恢复原有事务，并将当前正在进行的事务信息进行重置。下面我们看Spring如何开启一个新事务：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_72d271ed-a776-4ea1-aef5-1b0257bcb287.jpg)

这里我们看到了数据库连接的获取，如果是新事务需要获取新一个新的数据库连接，并为其设置了隔离级别、是否只读等属性，下面就是将事务信息记录到当前线程中：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_21873217-513b-4bf1-895f-8253406688d0.jpg)

接下来就是记录事务状态并返回事务信息：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_1e39eb93-a9fc-4bdb-bc75-78875f365d45.jpg)

然后就是我们目标业务方法的执行了，根据执行结果的不同做提交或回滚操作，我们先看一下回滚操作：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_bdee2ff7-6b22-4979-a4a6-68768cb07ec9.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_6efd682b-d826-42ec-a742-36664fa1ee55.jpg)

其中回滚条件默认为RuntimeException或Error，我们也可以自行配置。

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_49828188-3b75-4e8e-a27c-8d24efee1152.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_3ff0a0a4-7481-4769-a5f7-9b798376032a.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f8e84baa-f411-432d-b624-06c03d6a0770.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_65273017-9cb7-4b02-b4f3-704f28705ca7.jpg)

保存点一般用于嵌入式事务，内嵌事务的回滚不会引起外部事务的回滚。下面我们来看新事务的回滚：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_a539fba7-ac99-4be3-97d6-670191bf3795.jpg)

很简单，就是获取当前线程的数据库连接并调用其rollback方法进行回滚，使用的是底层数据库连接提供的API。最后还有一个清理和恢复挂起事务的操作：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_f91a16c5-a49b-48ab-81d5-4fe4689d9848.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_d97f1fb2-3878-482e-9fe4-7fd5c21a4035.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_d6df1fef-4455-400b-9054-cf58c8e6c109.jpg)

如果事务执行前有事务挂起，那么当前事务执行结束后需要将挂起的事务恢复，挂起事务时保存了原事务信息，重置了当前事务信息，所以恢复操作就是将当前的事务信息设置为之前保存的原事务信息。到这里事务的回滚操作就结束了，下面让我们来看事务的提交操作：

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_15a2a67f-8aed-4e0c-a0e7-1ee9c52f6daa.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_370f9de0-e3a6-4839-ae98-611ee1fe01eb.jpg)

在上文分析回滚流程中我们提到了如果当前事务不是独立的事务，也没有保存点，在回滚的时候只是设置一个回滚标记，由外部事务提交时统一进行整体事务的回滚。

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_57b414a2-718d-48a8-ab6f-21d15421f7cc.jpg)

![](http://misc.linkedkeeper.com/misc/img/blog/201711/linkedkeeper0_e44c8ad4-55ec-4953-bfce-6a3ddd741ce5.jpg)

提交操作也是很简单的调用数据库连接底层API的commit方法。

参考链接：

http://blog.163.com/asd_wll/blog/static/2103104020124801348674/

https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#spring-data-tier