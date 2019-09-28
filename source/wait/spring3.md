	---
title: Spring源码剖析3：懒加载的单例Bean获取过程分析
date: 2018-06-03 22:27:51
tags:
    - Spring
categories:
	- 后端
	- Spring
---
转自[【Spring源码分析】Bean加载流程概览](https://www.cnblogs.com/xrq730/p/6285358.html)



本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Spring源码解析 https://blog.csdn.net/column/details/21851.html

部分代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->

转自：http://www.cnblogs.com/xrq730

## 代码入口

之前写文章都会啰啰嗦嗦一大堆再开始，进入【Spring源码分析】这个板块就直接切入正题了。

很多朋友可能想看Spring源码，但是不知道应当如何入手去看，这个可以理解：Java开发者通常从事的都是Java Web的工作，对于程序员来说，一个Web项目用到Spring，只是配置一下配置文件而已，Spring的加载过程相对是不太透明的，不太好去找加载的代码入口。

下面有很简单的一段代码可以作为Spring代码加载的入口：



<pre> 1 ApplicationContext ac = new ClassPathXmlApplicationContext("spring.xml");
 2 ac.getBean(XXX.class);</pre>



ClassPathXmlApplicationContext用于加载CLASSPATH下的Spring配置文件，可以看到，第二行就已经可以获取到Bean的实例了，那么必然第一行就已经完成了对所有Bean实例的加载，因此可以通过ClassPathXmlApplicationContext作为入口。为了后面便于代码阅读，先给出一下ClassPathXmlApplicationContext这个类的继承关系：![](https://images2015.cnblogs.com/blog/801753/201702/801753-20170201125310058-568989522.png)

大致的继承关系是如上图所示的，由于版面的关系，没有继续画下去了，左下角的ApplicationContext应当还有一层继承关系，比较关键的一点是它是BeanFactory的子接口。

最后声明一下，本文使用的Spring版本为3.0.7，比较老，使用这个版本纯粹是因为公司使用而已。

ClassPathXmlApplicationContext存储内容

为了更理解ApplicationContext，拿一个实例ClassPathXmlApplicationContext举例，看一下里面存储的内容，加深对ApplicationContext的认识，以表格形式展现：

| 对象名 | 类  型 | 作  用 | 归属类 |
| --- | --- | --- | --- |
| configResources | Resource[] | 配置文件资源对象数组 | ClassPathXmlApplicationContext |
| configLocations | String[] | 配置文件字符串数组，存储配置文件路径 | AbstractRefreshableConfigApplicationContext |
| beanFactory | DefaultListableBeanFactory | 上下文使用的Bean工厂 | AbstractRefreshableApplicationContext |
| beanFactoryMonitor | Object | Bean工厂使用的同步监视器 | AbstractRefreshableApplicationContext |
| id | String | 上下文使用的唯一Id，标识此ApplicationContext | AbstractApplicationContext |
| parent | ApplicationContext | 父级ApplicationContext | AbstractApplicationContext |
| beanFactoryPostProcessors | List<BeanFactoryPostProcessor> | 存储BeanFactoryPostProcessor接口，Spring提供的一个扩展点 | AbstractApplicationContext |
| startupShutdownMonitor | Object | refresh方法和destory方法公用的一个监视器，避免两个方法同时执行 | AbstractApplicationContext |
| shutdownHook | Thread | Spring提供的一个钩子，JVM停止执行时会运行Thread里面的方法 | AbstractApplicationContext |
| resourcePatternResolver | ResourcePatternResolver | 上下文使用的资源格式解析器 | AbstractApplicationContext |
| lifecycleProcessor | LifecycleProcessor | 用于管理Bean生命周期的生命周期处理器接口 | AbstractApplicationContext |
| messageSource | MessageSource | 用于实现国际化的一个接口 | AbstractApplicationContext |
| applicationEventMulticaster | ApplicationEventMulticaster | Spring提供的事件管理机制中的事件多播器接口 | AbstractApplicationContext |
| applicationListeners | Set<ApplicationListener> | Spring提供的事件管理机制中的应用监听器 | AbstractApplicationContext |

ClassPathXmlApplicationContext构造函数

看下ClassPathXmlApplicationContext的构造函数：



<pre> 1 public ClassPathXmlApplicationContext(String configLocation) throws BeansException { 2     this(new String[] {configLocation}, true, null);
 3 }</pre>





![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre>1 public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) 2         throws BeansException { 3 
4     super(parent); 5 setConfigLocations(configLocations); 6     if (refresh) { 7 refresh(); 8 } 9 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



从第二段代码看，总共就做了三件事：

　　1、super(parent)

　　　　没什么太大的作用，设置一下父级ApplicationContext，这里是null

　　2、setConfigLocations(configLocations)

　　　　代码就不贴了，一看就知道，里面做了两件事情：

　　　　（1）将指定的Spring配置文件的路径存储到本地

　　　　（2）解析Spring配置文件路径中的${PlaceHolder}占位符，替换为系统变量中PlaceHolder对应的Value值，System本身就自带一些系统变量比如class.path、os.name、user.dir等，也可以通过System.setProperty()方法设置自己需要的系统变量

## refresh()

　　　　这个就是整个Spring Bean加载的核心了，它是ClassPathXmlApplicationContext的父类AbstractApplicationContext的一个方法，顾名思义，用于刷新整个Spring上下文信息，定义了整个Spring上下文加载的流程。

refresh方法

上面已经说了，refresh()方法是整个Spring Bean加载的核心，因此看一下整个refresh()方法的定义：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public void refresh() throws BeansException, IllegalStateException { 2         synchronized (this.startupShutdownMonitor) {
 3             // Prepare this context for refreshing.
 4             prepareRefresh();
 5 
 6             // Tell the subclass to refresh the internal bean factory.
 7             ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); 8 
 9             // Prepare the bean factory for use in this context.
10 prepareBeanFactory(beanFactory); 11 
12             try { 13                 // Allows post-processing of the bean factory in context subclasses.
14 postProcessBeanFactory(beanFactory); 15 
16                 // Invoke factory processors registered as beans in the context.
17 invokeBeanFactoryPostProcessors(beanFactory); 18 
19                 // Register bean processors that intercept bean creation.
20 registerBeanPostProcessors(beanFactory); 21 
22                 // Initialize message source for this context.
23 initMessageSource(); 24 
25                 // Initialize event multicaster for this context.
26 initApplicationEventMulticaster(); 27 
28                 // Initialize other special beans in specific context subclasses.
29 onRefresh(); 30 
31                 // Check for listener beans and register them.
32 registerListeners(); 33 
34                 // Instantiate all remaining (non-lazy-init) singletons.
35 finishBeanFactoryInitialization(beanFactory); 36 
37                 // Last step: publish corresponding event.
38 finishRefresh(); 39 } 40 
41             catch (BeansException ex) { 42                 // Destroy already created singletons to avoid dangling resources.
43 destroyBeans(); 44 
45                 // Reset 'active' flag.
46 cancelRefresh(ex); 47 
48                 // Propagate exception to caller.
49                 throw ex; 50 } 51 } 52     }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



每个子方法的功能之后一点一点再分析，首先refresh()方法有几点是值得我们学习的：

　　1、方法是加锁的，这么做的原因是避免多线程同时刷新Spring上下文

　　2、尽管加锁可以看到是针对整个方法体的，但是没有在方法前加synchronized关键字，而使用了对象锁startUpShutdownMonitor，这样做有两个好处：

　　　　（1）refresh()方法和close()方法都使用了startUpShutdownMonitor对象锁加锁，这就保证了在调用refresh()方法的时候无法调用close()方法，反之亦然，避免了冲突

　　　　（2）另外一个好处不在这个方法中体现，但是提一下，使用对象锁可以减小了同步的范围，只对不能并发的代码块进行加锁，提高了整体代码运行的效率

　　3、方法里面使用了每个子方法定义了整个refresh()方法的流程，使得整个方法流程清晰易懂。这点是非常值得学习的，一个方法里面几十行甚至上百行代码写在一起，在我看来会有三个显著的问题：

　　　　（1）扩展性降低。反过来讲，假使把流程定义为方法，子类可以继承父类，可以根据需要重写方法

　　　　（2）代码可读性差。很简单的道理，看代码的人是愿意看一段500行的代码，还是愿意看10段50行的代码？

　　　　（3）代码可维护性差。这点和上面的类似但又有不同，可维护性差的意思是，一段几百行的代码，功能点不明确，不易后人修改，可能会导致“牵一发而动全身”

prepareRefresh方法

下面挨个看refresh方法中的子方法，首先是prepareRefresh方法，看一下源码：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 /**
 2  * Prepare this context for refreshing, setting its startup date and
 3  * active flag.
 4  */
 5 protected void prepareRefresh() { 6     this.startupDate = System.currentTimeMillis(); 7         synchronized (this.activeMonitor) {
 8         this.active = true;
 9 } 10 
11     if (logger.isInfoEnabled()) { 12         logger.info("Refreshing " + this); 13 } 14 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



这个方法功能比较简单，顾名思义，准备刷新Spring上下文，其功能注释上写了：

1、设置一下刷新Spring上下文的开始时间

2、将active标识位设置为true

另外可以注意一下12行这句日志，这句日志打印了真正加载Spring上下文的Java类。

obtainFreshBeanFactory方法

obtainFreshBeanFactory方法的作用是获取刷新Spring上下文的Bean工厂，其代码实现为：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre>1 protected ConfigurableListableBeanFactory obtainFreshBeanFactory() { 2 refreshBeanFactory(); 3     ConfigurableListableBeanFactory beanFactory = getBeanFactory(); 4     if (logger.isDebugEnabled()) { 5         logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory); 6 } 7     return beanFactory; 8 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



其核心是第二行的refreshBeanFactory方法，这是一个抽象方法，有AbstractRefreshableApplicationContext和GenericApplicationContext这两个子类实现了这个方法，看一下上面ClassPathXmlApplicationContext的继承关系图即知，调用的应当是AbstractRefreshableApplicationContext中实现的refreshBeanFactory，其源码为：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected final void refreshBeanFactory() throws BeansException { 2     if (hasBeanFactory()) { 3         destroyBeans();
 4         closeBeanFactory();
 5     }
 6     try { 7         DefaultListableBeanFactory beanFactory = createBeanFactory(); 8         beanFactory.setSerializationId(getId());
 9 customizeBeanFactory(beanFactory); 10 loadBeanDefinitions(beanFactory); 11         synchronized (this.beanFactoryMonitor) { 12             this.beanFactory = beanFactory; 13 } 14 } 15     catch (IOException ex) { 16         throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex); 17 } 18 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



这段代码的核心是第7行，这行点出了DefaultListableBeanFactory这个类，这个类是构造Bean的核心类，这个类的功能会在下一篇文章中详细解读，首先给出DefaultListableBeanFactory的继承关系图：

![](https://images2015.cnblogs.com/blog/801753/201702/801753-20170202113720339-1992987568.png)

AbstractAutowireCapableBeanFactory这个类的继承层次比较深，版面有限，就没有继续画下去了，本图基本上清楚地展示了DefaultListableBeanFactory的层次结构。

为了更清晰地说明DefaultListableBeanFactory的作用，列举一下DefaultListableBeanFactory中存储的一些重要对象及对象中的内容，DefaultListableBeanFactory基本就是操作这些对象，以表格形式说明：

|  对象名 | 类  型 |  作    用 | 归属类 |
| --- | --- | --- | --- |
|  aliasMap | Map<String, String> | 存储Bean名称->Bean别名映射关系  |  SimpleAliasRegistry |
| singletonObjects  | Map<String, Object> |  存储单例Bean名称->单例Bean实现映射关系 | DefaultSingletonBeanRegistry  |
|  singletonFactories |  Map<String, ObjectFactory> | 存储Bean名称->ObjectFactory实现映射关系  | DefaultSingletonBeanRegistry  |
| earlySingletonObjects  |  Map<String, Object> | 存储Bean名称->预加载Bean实现映射关系   |  DefaultSingletonBeanRegistry  |
| registeredSingletons  | Set<String>  | 存储注册过的Bean名 |  DefaultSingletonBeanRegistry  |
| singletonsCurrentlyInCreation  | Set<String> | 存储当前正在创建的Bean名  |   DefaultSingletonBeanRegistry   |
|  disposableBeans |  Map<String, Object> | 存储Bean名称->Disposable接口实现Bean实现映射关系   |    DefaultSingletonBeanRegistry    |
|  factoryBeanObjectCache |  Map<String, Object> | 存储Bean名称->FactoryBean接口Bean实现映射关系 | FactoryBeanRegistrySupport  |
| propertyEditorRegistrars  |  Set<PropertyEditorRegistrar> | 存储PropertyEditorRegistrar接口实现集合 | AbstractBeanFactory  |
|  embeddedValueResolvers | List<StringValueResolver>  | 存储StringValueResolver（字符串解析器）接口实现列表 | AbstractBeanFactory  |
| beanPostProcessors  | List<BeanPostProcessor>  | 存储 BeanPostProcessor接口实现列表 | AbstractBeanFactory |
| mergedBeanDefinitions  | Map<String, RootBeanDefinition>  | 存储Bean名称->合并过的根Bean定义映射关系  | AbstractBeanFactory  |
|  alreadyCreated | Set<String>  | 存储至少被创建过一次的Bean名集合  |  AbstractBeanFactory   |
| ignoredDependencyInterfaces  | Set<Class>  | 存储不自动装配的接口Class对象集合  | AbstractAutowireCapableBeanFactory  |
|  resolvableDependencies | Map<Class, Object>  | 存储修正过的依赖映射关系  | DefaultListableBeanFactory  |
| beanDefinitionMap  | Map<String, BeanDefinition>  | 存储Bean名称-->Bean定义映射关系  | DefaultListableBeanFactory   |
| beanDefinitionNames | List<String> | 存储Bean定义名称列表  |  DefaultListableBeanFactory   |



================================================================================== 








## Spring是如何初始化Bean实例对象

代码入口

上文[【Spring源码分析】Bean加载流程概览](http://www.cnblogs.com/xrq730/p/6285358.html)，比较详细地分析了Spring上下文加载的代码入口，并且在AbstractApplicationContext的refresh方法中，点出了finishBeanFactoryInitialization方法完成了对于所有非懒加载的Bean的初始化。

finishBeanFactoryInitialization方法中调用了DefaultListableBeanFactory的preInstantiateSingletons方法，本文针对preInstantiateSingletons进行分析，解读一下Spring是如何初始化Bean实例对象出来的。

DefaultListableBeanFactory的preInstantiateSingletons方法

DefaultListableBeanFactory的preInstantiateSingletons方法，顾名思义，初始化所有的单例Bean，看一下方法的定义：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public void preInstantiateSingletons() throws BeansException { 2     if (this.logger.isInfoEnabled()) {
 3         this.logger.info("Pre-instantiating singletons in " + this);
 4     }
 5     synchronized (this.beanDefinitionMap) {
 6         // Iterate over a copy to allow for init methods which in turn register new bean definitions. 7         // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
 8         List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
 9         for (String beanName : beanNames) { 10             RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName); 11             if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) { 12                 if (isFactoryBean(beanName)) { 13                     final FactoryBean factory = (FactoryBean) getBean(FACTORY_BEAN_PREFIX + beanName); 14                     boolean isEagerInit; 15                     if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) { 16                         isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() { 17                             public Boolean run() { 18                                 return ((SmartFactoryBean) factory).isEagerInit(); 19 } 20 }, getAccessControlContext()); 21 } 22                     else { 23                         isEagerInit = (factory instanceof SmartFactoryBean &&
24 ((SmartFactoryBean) factory).isEagerInit()); 25 } 26                     if (isEagerInit) { 27 getBean(beanName); 28 } 29 } 30                 else { 31 getBean(beanName); 32 } 33 } 34 } 35 } 36 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



这里先解释一下getMergedLocalBeanDefinition方法的含义，因为这个方法会常常看到。Bean定义公共的抽象类是AbstractBeanDefinition，普通的Bean在Spring加载Bean定义的时候，实例化出来的是GenericBeanDefinition，而Spring上下文包括实例化所有Bean用的AbstractBeanDefinition是RootBeanDefinition，这时候就使用getMergedLocalBeanDefinition方法做了一次转化，将非RootBeanDefinition转换为RootBeanDefinition以供后续操作。

解释完了getMergedLocalBeanDefinition方法的作用，第1行~第10行的代码就没什么好说的了，根据beanName拿到RootBeanDefinition而已。由于此方法实例化的是所有非懒加载的单例Bean，因此要实例化Bean，必须满足11行的三个定义：

（1）不是抽象的

（2）必须是单例的

（3）必须是非懒加载的

接着简单看一下第12行~第29行的代码，这段代码主要做的是一件事情：首先判断一下Bean是否FactoryBean的实现，接着判断Bean是否SmartFactoryBean的实现，假如Bean是SmartFactoryBean的实现并且eagerInit（这个单词字面意思是渴望加载，找不到一个好的词语去翻译，意思就是定义了这个Bean需要立即加载的意思）的话，会立即实例化这个Bean。Java开发人员不需要关注这段代码，因为SmartFactoryBean基本不会用到，我翻译一下Spring官网对于SmartFactoryBean的定义描述：

*   FactoryBean接口的扩展接口。接口实现并不表示是否总是返回单独的实例对象，比如FactoryBean.isSingleton()实现返回false的情况并不清晰地表示每次返回的都是单独的实例对象
*   不实现这个扩展接口的简单FactoryBean的实现，FactoryBean.isSingleton()实现返回false总是简单地告诉我们每次返回的都是单独的实例对象，暴露出来的对象只能够通过命令访问
*   注意：这个接口是一个有特殊用途的接口，主要用于框架内部使用与Spring相关。通常，应用提供的FactoryBean接口实现应当只需要实现简单的FactoryBean接口即可，新方法应当加入到扩展接口中去

代码示例

为了后面的代码分析方便，事先我定义一个Bean：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 package org.xrq.action; 2 
 3 import org.springframework.beans.factory.BeanClassLoaderAware; 4 import org.springframework.beans.factory.BeanNameAware; 5 import org.springframework.beans.factory.InitializingBean; 6 
 7 public class MultiFunctionBean implements InitializingBean, BeanNameAware, BeanClassLoaderAware { 8 
 9     private int propertyA; 10     
11     private int propertyB; 12     
13     public int getPropertyA() { 14         return propertyA; 15 } 16 
17     public void setPropertyA(int propertyA) { 18         this.propertyA = propertyA; 19 } 20 
21     public int getPropertyB() { 22         return propertyB; 23 } 24 
25     public void setPropertyB(int propertyB) { 26         this.propertyB = propertyB; 27 } 28     
29     public void initMethod() { 30         System.out.println("Enter MultiFunctionBean.initMethod()"); 31 } 32 
33 @Override 34     public void setBeanClassLoader(ClassLoader classLoader) { 35         System.out.println("Enter MultiFunctionBean.setBeanClassLoader(ClassLoader classLoader)"); 36 } 37 
38 @Override 39     public void setBeanName(String name) { 40         System.out.println("Enter MultiFunctionBean.setBeanName(String name)"); 41 } 42 
43 @Override 44     public void afterPropertiesSet() throws Exception { 45         System.out.println("Enter MultiFunctionBean.afterPropertiesSet()"); 46 } 47     
48 @Override 49     public String toString() { 50         return "MultiFunctionBean [propertyA=" + propertyA + ", propertyB=" + propertyB + "]"; 51 } 52     
53 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



定义对应的spring.xml：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre>1 <?xml version="1.0" encoding="UTF-8"?>
2 <beans xmlns="http://www.springframework.org/schema/beans"
3     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
4     xsi:schemaLocation="http://www.springframework.org/schema/beans 5     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
6     
7     <bean id="multiFunctionBean" class="org.xrq.action.MultiFunctionBean" init-method="initMethod" />
8     
9 </beans></pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



利用这个MultiFunctionBean，我们可以用来探究Spring加载Bean的多种机制。

doGetBean方法构造Bean流程

上面把getBean之外的代码都分析了一下，看代码就可以知道，获取Bean对象实例，都是通过getBean方法，getBean方法最终调用的是DefaultListableBeanFactory的父类AbstractBeanFactory类的doGetBean方法，因此这部分重点分析一下doGetBean方法是如何构造出一个单例的Bean的。


## doGetBean方法是如何构造出一个单例的Bean
看一下doGetBean方法的代码实现，比较长：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected <T> T doGetBean( 2         final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) 3         throws BeansException { 4 
 5     final String beanName = transformedBeanName(name); 6     Object bean;
 7 
 8     // Eagerly check singleton cache for manually registered singletons.
 9     Object sharedInstance = getSingleton(beanName); 10     if (sharedInstance != null && args == null) {
 11         if (logger.isDebugEnabled()) { 12             if (isSingletonCurrentlyInCreation(beanName)) { 13                 logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
 14                         "' that is not fully initialized yet - a consequence of a circular reference");
 15             }
 16             else { 17                 logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
 18             }
 19         }
 20         bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
 21     }
 22 
 23     else { 24         // Fail if we're already creating this bean instance: 25         // We're assumably within a circular reference.
 26         if (isPrototypeCurrentlyInCreation(beanName)) { 27             throw new BeanCurrentlyInCreationException(beanName); 28         }
 29 
 30         // Check if bean definition exists in this factory.
 31         BeanFactory parentBeanFactory = getParentBeanFactory(); 32         if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
 33             // Not found -> check parent.
 34             String nameToLookup = originalBeanName(name); 35             if (args != null) {
 36                 // Delegation to parent with explicit args.
 37                 return (T) parentBeanFactory.getBean(nameToLookup, args); 38             }
 39             else { 40                 // No args -> delegate to standard getBean method.
 41                 return parentBeanFactory.getBean(nameToLookup, requiredType); 42             }
 43         }
 44 
 45         if (!typeCheckOnly) {
 46             markBeanAsCreated(beanName);
 47         }
 48 
 49         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName); 50         checkMergedBeanDefinition(mbd, beanName, args);
 51 
 52         // Guarantee initialization of beans that the current bean depends on.
 53         String[] dependsOn = mbd.getDependsOn(); 54         if (dependsOn != null) {
 55             for (String dependsOnBean : dependsOn) { 56                 getBean(dependsOnBean);
 57                 registerDependentBean(dependsOnBean, beanName);
 58             }
 59         }
 60 
 61         // Create bean instance.
 62         if (mbd.isSingleton()) { 63             sharedInstance = getSingleton(beanName, new ObjectFactory() { 64                 public Object getObject() throws BeansException { 65                     try { 66                         return createBean(beanName, mbd, args); 67                     }
 68                     catch (BeansException ex) { 69                         // Explicitly remove instance from singleton cache: It might have been put there 70                         // eagerly by the creation process, to allow for circular reference resolution. 71                         // Also remove any beans that received a temporary reference to the bean.
 72                         destroySingleton(beanName);
 73                         throw ex; 74                     }
 75                 }
 76             });
 77             bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd); 78         }
 79 
 80         else if (mbd.isPrototype()) { 81             // It's a prototype -> create a new instance.
 82             Object prototypeInstance = null;
 83             try { 84                 beforePrototypeCreation(beanName);
 85                 prototypeInstance = createBean(beanName, mbd, args); 86             }
 87             finally { 88                 afterPrototypeCreation(beanName);
 89             }
 90             bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd); 91         }
 92 
 93         else { 94             String scopeName = mbd.getScope(); 95             final Scope scope = this.scopes.get(scopeName);
 96             if (scope == null) {
 97                 throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
 98             }
 99             try { 100                 Object scopedInstance = scope.get(beanName, new ObjectFactory() { 101                     public Object getObject() throws BeansException { 102 beforePrototypeCreation(beanName); 103                         try { 104                             return createBean(beanName, mbd, args); 105 } 106                         finally { 107 afterPrototypeCreation(beanName); 108 } 109 } 110 }); 111                 bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd); 112 } 113             catch (IllegalStateException ex) { 114                 throw new BeanCreationException(beanName, 115                         "Scope '" + scopeName + "' is not active for the current thread; " +
116                         "consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", 117 ex); 118 } 119 } 120 } 121 
122     // Check if required type matches the type of the actual bean instance.
123     if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) { 124         try { 125             return getTypeConverter().convertIfNecessary(bean, requiredType); 126 } 127         catch (TypeMismatchException ex) { 128             if (logger.isDebugEnabled()) { 129                 logger.debug("Failed to convert bean '" + name + "' to required type [" +
130                         ClassUtils.getQualifiedName(requiredType) + "]", ex); 131 } 132             throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass()); 133 } 134 } 135     return (T) bean; 136 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



首先第9行~第21行的代码，第9行的代码就不进去看了，简单说一下：首先检查一下本地的单例缓存是否已经加载过Bean，没有的话再检查earlySingleton缓存是否已经加载过Bean（又是early，不好找到词语翻译），没有的话执行后面的逻辑。

接着第26行~第50行，这里执行的都是一些基本的检查和简单的操作，包括bean是否是prototype的（prototype的Bean当前创建会抛出异常）、是否抽象的、将beanName加入alreadyCreated这个Set中等。 

接着第53行~第59行，我们经常在bean标签中看到depends-on这个属性，就是通过这段保证了depends-on依赖的Bean会优先于当前Bean被加载。 

接着第62行~第78行、第80行~第91行、第93行~第120行有三个判断，显然上面的MultiFunctionBean是一个单例的Bean也是本文探究的重点，因此执行第62行~第78行的逻辑。getSingleton方法不贴了，有一些前置的判断，很简单的逻辑，重点就是调用了ObjectFactory的getObject()方法来获取到单例Bean对象，方法的实现是调用了createBean方法，createBean方法是AbstractBeanFactory的子类AbstractAutowireCapableBeanFactory的一个方法，看一下它的方法实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) 2         throws BeanCreationException { 3 
 4     if (logger.isDebugEnabled()) { 5         logger.debug("Creating instance of bean '" + beanName + "'");
 6     }
 7     // Make sure bean class is actually resolved at this point.
 8     resolveBeanClass(mbd, beanName);
 9 
10     // Prepare method overrides.
11     try { 12 mbd.prepareMethodOverrides(); 13 } 14     catch (BeanDefinitionValidationException ex) { 15         throw new BeanDefinitionStoreException(mbd.getResourceDescription(), 16                 beanName, "Validation of method overrides failed", ex); 17 } 18 
19     try { 20         // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
21         Object bean = resolveBeforeInstantiation(beanName, mbd); 22         if (bean != null) { 23             return bean; 24 } 25 } 26     catch (Throwable ex) { 27         throw new BeanCreationException(mbd.getResourceDescription(), beanName, 28                 "BeanPostProcessor before instantiation of bean failed", ex); 29 } 30 
31     Object beanInstance = doCreateBean(beanName, mbd, args); 32     if (logger.isDebugEnabled()) { 33         logger.debug("Finished creating instance of bean '" + beanName + "'"); 34 } 35     return beanInstance; 36 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



前面的代码都没什么意义，代码执行到第31行：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) { 2     // Instantiate the bean.
 3     BeanWrapper instanceWrapper = null;
 4     if (mbd.isSingleton()) { 5         instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
 6     }
 7     if (instanceWrapper == null) {
 8         instanceWrapper = createBeanInstance(beanName, mbd, args); 9 } 10     final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null); 11     Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null); 12 
13     // Allow post-processors to modify the merged bean definition.
14     synchronized (mbd.postProcessingLock) { 15         if (!mbd.postProcessed) { 16 applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName); 17             mbd.postProcessed = true; 18 } 19 } 20 
21     // Eagerly cache singletons to be able to resolve circular references 22     // even when triggered by lifecycle interfaces like BeanFactoryAware.
23     boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
24 isSingletonCurrentlyInCreation(beanName)); 25     if (earlySingletonExposure) { 26         if (logger.isDebugEnabled()) { 27             logger.debug("Eagerly caching bean '" + beanName +
28                     "' to allow for resolving potential circular references"); 29 } 30         addSingletonFactory(beanName, new ObjectFactory() { 31             public Object getObject() throws BeansException { 32                 return getEarlyBeanReference(beanName, mbd, bean); 33 } 34 }); 35 } 36 
37     // Initialize the bean instance.
38     Object exposedObject = bean; 39     try { 40 populateBean(beanName, mbd, instanceWrapper); 41         if (exposedObject != null) { 42             exposedObject = initializeBean(beanName, exposedObject, mbd); 43 } 44 } 45     catch (Throwable ex) { 46         if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) { 47             throw (BeanCreationException) ex; 48 } 49         else { 50             throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex); 51 } 52 } 53 
54     if (earlySingletonExposure) { 55         Object earlySingletonReference = getSingleton(beanName, false); 56         if (earlySingletonReference != null) { 57             if (exposedObject == bean) { 58                 exposedObject = earlySingletonReference; 59 } 60             else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) { 61                 String[] dependentBeans = getDependentBeans(beanName); 62                 Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length); 63                 for (String dependentBean : dependentBeans) { 64                     if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) { 65 actualDependentBeans.add(dependentBean); 66 } 67 } 68                 if (!actualDependentBeans.isEmpty()) { 69                     throw new BeanCurrentlyInCreationException(beanName, 70                             "Bean with name '" + beanName + "' has been injected into other beans [" +
71                                 StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
72                             "] in its raw version as part of a circular reference, but has eventually been " +
73                             "wrapped. This means that said other beans do not use the final version of the " +
74                             "bean. This is often the result of over-eager type matching - consider using " +
75                             "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example."); 76 } 77 } 78 } 79 } 80 
81     // Register bean as disposable.
82     try { 83 registerDisposableBeanIfNecessary(beanName, bean, mbd); 84 } 85     catch (BeanDefinitionValidationException ex) { 86         throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex); 87 } 88 
89     return exposedObject; 90 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)

## doCreateBean方法

代码跟踪到这里，已经到了主流程，接下来分段分析doCreateBean方法的代码。

创建Bean实例

第8行的createBeanInstance方法，会创建出Bean的实例，并包装为BeanWrapper，看一下createBeanInstance方法，只贴最后一段比较关键的：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 // Need to determine the constructor...
 2 Constructor[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName); 3 if (ctors != null ||
 4         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
 5         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
 6     return autowireConstructor(beanName, mbd, ctors, args); 7 }
 8 
 9 // No special handling: simply use no-arg constructor.
10 return instantiateBean(beanName, mbd);</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



意思是bean标签使用构造函数注入属性的话，执行第6行，否则执行第10行。MultiFunctionBean使用默认构造函数，使用setter注入属性，因此执行第10行代码：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) { 2     try { 3         Object beanInstance;
 4         final BeanFactory parent = this;
 5         if (System.getSecurityManager() != null) {
 6             beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
 7                 public Object run() { 8                     return getInstantiationStrategy().instantiate(mbd, beanName, parent); 9 } 10 }, getAccessControlContext()); 11 } 12         else { 13             beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent); 14 } 15         BeanWrapper bw = new BeanWrapperImpl(beanInstance); 16 initBeanWrapper(bw); 17         return bw; 18 } 19     catch (Throwable ex) { 20         throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex); 21 } 22 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



代码执行到13行：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public Object instantiate(RootBeanDefinition beanDefinition, String beanName, BeanFactory owner) { 2     // Don't override the class with CGLIB if no overrides.
 3     if (beanDefinition.getMethodOverrides().isEmpty()) { 4         Constructor<?> constructorToUse; 5         synchronized (beanDefinition.constructorArgumentLock) { 6             constructorToUse = (Constructor<?>) beanDefinition.resolvedConstructorOrFactoryMethod;
 7             if (constructorToUse == null) {
 8                 final Class clazz = beanDefinition.getBeanClass(); 9                 if (clazz.isInterface()) { 10                     throw new BeanInstantiationException(clazz, "Specified class is an interface"); 11 } 12                 try { 13                     if (System.getSecurityManager() != null) { 14                         constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor>() { 15                             public Constructor run() throws Exception { 16                                 return clazz.getDeclaredConstructor((Class[]) null); 17 } 18 }); 19 } 20                     else { 21                         constructorToUse = clazz.getDeclaredConstructor((Class[]) null); 22 } 23                     beanDefinition.resolvedConstructorOrFactoryMethod = constructorToUse; 24 } 25                 catch (Exception ex) { 26                     throw new BeanInstantiationException(clazz, "No default constructor found", ex); 27 } 28 } 29 } 30         return BeanUtils.instantiateClass(constructorToUse); 31 } 32     else { 33         // Must generate CGLIB subclass.
34         return instantiateWithMethodInjection(beanDefinition, beanName, owner); 35 } 36 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



整段代码都在做一件事情，就是选择一个使用的构造函数。当然第9行顺带做了一个判断：实例化一个接口将报错。

最后调用到30行，看一下代码：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException { 2     Assert.notNull(ctor, "Constructor must not be null");
 3     try { 4         ReflectionUtils.makeAccessible(ctor);
 5         return ctor.newInstance(args); 6     }
 7     catch (InstantiationException ex) { 8         throw new BeanInstantiationException(ctor.getDeclaringClass(), 9                 "Is it an abstract class?", ex); 10 } 11     catch (IllegalAccessException ex) { 12         throw new BeanInstantiationException(ctor.getDeclaringClass(), 13                 "Is the constructor accessible?", ex); 14 } 15     catch (IllegalArgumentException ex) { 16         throw new BeanInstantiationException(ctor.getDeclaringClass(), 17                 "Illegal arguments for constructor", ex); 18 } 19     catch (InvocationTargetException ex) { 20         throw new BeanInstantiationException(ctor.getDeclaringClass(), 21                 "Constructor threw exception", ex.getTargetException()); 22 } 23 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



通过反射生成Bean的实例。看到前面有一步makeAccessible，这意味着即使Bean的构造函数是private、protected的，依然不影响Bean的构造。

最后注意一下，这里被实例化出来的Bean并不会直接返回，而是会被包装为BeanWrapper继续在后面使用。





## doCreateBean方法

上文[【Spring源码分析】非懒加载的单例Bean初始化过程（上篇）](http://www.cnblogs.com/xrq730/p/6361578.html)，分析了单例的Bean初始化流程，并跟踪代码进入了主流程，看到了Bean是如何被实例化出来的。先贴一下AbstractAutowireCapableBeanFactory的doCreateBean方法代码：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) { 2     // Instantiate the bean.
 3     BeanWrapper instanceWrapper = null;
 4     if (mbd.isSingleton()) { 5         instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
 6     }
 7     if (instanceWrapper == null) {
 8         instanceWrapper = createBeanInstance(beanName, mbd, args); 9 } 10     final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null); 11     Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null); 12 
13     // Allow post-processors to modify the merged bean definition.
14     synchronized (mbd.postProcessingLock) { 15         if (!mbd.postProcessed) { 16 applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName); 17             mbd.postProcessed = true; 18 } 19 } 20 
21     // Eagerly cache singletons to be able to resolve circular references 22     // even when triggered by lifecycle interfaces like BeanFactoryAware.
23     boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
24 isSingletonCurrentlyInCreation(beanName)); 25     if (earlySingletonExposure) { 26         if (logger.isDebugEnabled()) { 27             logger.debug("Eagerly caching bean '" + beanName +
28                     "' to allow for resolving potential circular references"); 29 } 30         addSingletonFactory(beanName, new ObjectFactory() { 31             public Object getObject() throws BeansException { 32                 return getEarlyBeanReference(beanName, mbd, bean); 33 } 34 }); 35 } 36 
37     // Initialize the bean instance.
38     Object exposedObject = bean; 39     try { 40 populateBean(beanName, mbd, instanceWrapper); 41         if (exposedObject != null) { 42             exposedObject = initializeBean(beanName, exposedObject, mbd); 43 } 44 } 45     catch (Throwable ex) { 46         if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) { 47             throw (BeanCreationException) ex; 48 } 49         else { 50             throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex); 51 } 52 } 53 
54     if (earlySingletonExposure) { 55         Object earlySingletonReference = getSingleton(beanName, false); 56         if (earlySingletonReference != null) { 57             if (exposedObject == bean) { 58                 exposedObject = earlySingletonReference; 59 } 60             else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) { 61                 String[] dependentBeans = getDependentBeans(beanName); 62                 Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length); 63                 for (String dependentBean : dependentBeans) { 64                     if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) { 65 actualDependentBeans.add(dependentBean); 66 } 67 } 68                 if (!actualDependentBeans.isEmpty()) { 69                     throw new BeanCurrentlyInCreationException(beanName, 70                             "Bean with name '" + beanName + "' has been injected into other beans [" +
71                             StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
72                             "] in its raw version as part of a circular reference, but has eventually been " +
73                             "wrapped. This means that said other beans do not use the final version of the " +
74                             "bean. This is often the result of over-eager type matching - consider using " +
75                             "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example."); 76 } 77 } 78 } 79 } 80 
81     // Register bean as disposable.
82     try { 83 registerDisposableBeanIfNecessary(beanName, bean, mbd); 84 } 85     catch (BeanDefinitionValidationException ex) { 86         throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex); 87 } 88 
89     return exposedObject; 90 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



下面继续分析初始化一个Bean的流程，不太重要的流程就跳过了。

属性注入

属性注入的代码比较好找，可以看一下40行，取名为populateBean，即填充Bean的意思，看一下代码实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected void populateBean(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw) { 2     PropertyValues pvs = mbd.getPropertyValues(); 3 
 4     if (bw == null) {
 5         if (!pvs.isEmpty()) {
 6             throw new BeanCreationException( 7                     mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
 8         }
 9         else { 10             // Skip property population phase for null instance.
11             return; 12 } 13 } 14 
15     // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the 16     // state of the bean before properties are set. This can be used, for example, 17     // to support styles of field injection.
18     boolean continueWithPropertyPopulation = true; 19 
20     if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) { 21         for (BeanPostProcessor bp : getBeanPostProcessors()) { 22             if (bp instanceof InstantiationAwareBeanPostProcessor) { 23                 InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp; 24                 if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) { 25                     continueWithPropertyPopulation = false; 26                     break; 27 } 28 } 29 } 30 } 31 
32     if (!continueWithPropertyPopulation) { 33         return; 34 } 35 
36     if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
37             mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) { 38         MutablePropertyValues newPvs = new MutablePropertyValues(pvs); 39 
40         // Add property values based on autowire by name if applicable.
41         if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) { 42 autowireByName(beanName, mbd, bw, newPvs); 43 } 44 
45         // Add property values based on autowire by type if applicable.
46         if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) { 47 autowireByType(beanName, mbd, bw, newPvs); 48 } 49 
50         pvs = newPvs; 51 } 52 
53     boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors(); 54     boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE); 55 
56     if (hasInstAwareBpps || needsDepCheck) { 57         PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw); 58         if (hasInstAwareBpps) { 59             for (BeanPostProcessor bp : getBeanPostProcessors()) { 60                 if (bp instanceof InstantiationAwareBeanPostProcessor) { 61                     InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp; 62                     pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName); 63                     if (pvs == null) { 64                         return; 65 } 66 } 67 } 68 } 69         if (needsDepCheck) { 70 checkDependencies(beanName, mbd, filteredPds, pvs); 71 } 72 } 73 
74 applyPropertyValues(beanName, mbd, bw, pvs); 75 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



这段代码层次有点深，跟一下74行的applyPropertyValues方法，最后那个pvs的实现类为MutablePropertyValues，里面持有一个List<PropertyValue>，每一个PropertyValue包含了此Bean属性的属性名与属性值。74行的代码实现为：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) { 2     if (pvs == null || pvs.isEmpty()) { 3         return;
 4     }
 5 
 6     MutablePropertyValues mpvs = null;
 7     List<PropertyValue> original; 8         
 9     if (System.getSecurityManager()!= null) { 10         if (bw instanceof BeanWrapperImpl) { 11 ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext()); 12 } 13 } 14 
15     if (pvs instanceof MutablePropertyValues) { 16         mpvs = (MutablePropertyValues) pvs; 17         if (mpvs.isConverted()) { 18             // Shortcut: use the pre-converted values as-is.
19             try { 20 bw.setPropertyValues(mpvs); 21                 return; 22 } 23             catch (BeansException ex) { 24                 throw new BeanCreationException( 25                         mbd.getResourceDescription(), beanName, "Error setting property values", ex); 26 } 27 } 28         original = mpvs.getPropertyValueList(); 29 } 30     else { 31         original = Arrays.asList(pvs.getPropertyValues()); 32 } 33 
34     TypeConverter converter = getCustomTypeConverter(); 35     if (converter == null) { 36         converter = bw; 37 } 38     BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter); 39 
40     // Create a deep copy, resolving any references for values.
41     List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size()); 42     boolean resolveNecessary = false; 43     for (PropertyValue pv : original) { 44         if (pv.isConverted()) { 45 deepCopy.add(pv); 46 } 47         else { 48             String propertyName = pv.getName(); 49             Object originalValue = pv.getValue(); 50             Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue); 51             Object convertedValue = resolvedValue; 52             boolean convertible = bw.isWritableProperty(propertyName) &&
53                         !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName); 54             if (convertible) { 55                 convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter); 56 } 57             // Possibly store converted value in merged bean definition, 58             // in order to avoid re-conversion for every created bean instance.
59             if (resolvedValue == originalValue) { 60                 if (convertible) { 61 pv.setConvertedValue(convertedValue); 62 } 63 deepCopy.add(pv); 64 } 65             else if (convertible && originalValue instanceof TypedStringValue &&
66                     !((TypedStringValue) originalValue).isDynamic() &&
67                     !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) { 68 pv.setConvertedValue(convertedValue); 69 deepCopy.add(pv); 70 } 71             else { 72                 resolveNecessary = true; 73                 deepCopy.add(new PropertyValue(pv, convertedValue)); 74 } 75 } 76 } 77     if (mpvs != null && !resolveNecessary) { 78 mpvs.setConverted(); 79 } 80 
81     // Set our (possibly massaged) deep copy.
82     try { 83         bw.setPropertyValues(new MutablePropertyValues(deepCopy)); 84 } 85     catch (BeansException ex) { 86         throw new BeanCreationException( 87                 mbd.getResourceDescription(), beanName, "Error setting property values", ex); 88 } 89 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



之后在第41行~第76行做了一次深拷贝（只是名字叫做深拷贝而已，其实就是遍历PropertyValue然后一个一个赋值到一个新的List而不是Java语义上的Clone，这里使用深拷贝是为了解析Values值中的所有引用），将PropertyValue一个一个赋值到一个新的List里面去，起名为deepCopy。最后执行83行进行复制，bw即BeanWrapper，持有Bean实例的一个Bean包装类，看一下代码实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid) 2         throws BeansException { 3 
 4     List<PropertyAccessException> propertyAccessExceptions = null;
 5     List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues ?
 6             ((MutablePropertyValues) pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));
 7     for (PropertyValue pv : propertyValues) { 8         try { 9             // This method may throw any BeansException, which won't be caught 10             // here, if there is a critical failure such as no matching field. 11             // We can attempt to deal only with less serious exceptions.
12 setPropertyValue(pv); 13 } 14         catch (NotWritablePropertyException ex) { 15             if (!ignoreUnknown) { 16                 throw ex; 17 } 18             // Otherwise, just ignore it and continue...
19 } 20         catch (NullValueInNestedPathException ex) { 21             if (!ignoreInvalid) { 22                 throw ex; 23 } 24             // Otherwise, just ignore it and continue...
25 } 26         catch (PropertyAccessException ex) { 27             if (propertyAccessExceptions == null) { 28                 propertyAccessExceptions = new LinkedList<PropertyAccessException>(); 29 } 30 propertyAccessExceptions.add(ex); 31 } 32 } 33 
34     // If we encountered individual exceptions, throw the composite exception.
35     if (propertyAccessExceptions != null) { 36         PropertyAccessException[] paeArray =
37                 propertyAccessExceptions.toArray(new PropertyAccessException[propertyAccessExceptions.size()]); 38         throw new PropertyBatchUpdateException(paeArray); 39 } 40 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



这段代码没什么特别的，遍历前面的deepCopy，拿每一个PropertyValue，执行第12行的setPropertyValue：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public void setPropertyValue(PropertyValue pv) throws BeansException { 2     PropertyTokenHolder tokens = (PropertyTokenHolder) pv.resolvedTokens; 3     if (tokens == null) {
 4         String propertyName = pv.getName(); 5         BeanWrapperImpl nestedBw;
 6         try { 7             nestedBw = getBeanWrapperForPropertyPath(propertyName); 8         }
 9         catch (NotReadablePropertyException ex) { 10             throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName, 11                     "Nested property in path '" + propertyName + "' does not exist", ex); 12 } 13         tokens = getPropertyNameTokens(getFinalPath(nestedBw, propertyName)); 14         if (nestedBw == this) { 15             pv.getOriginalPropertyValue().resolvedTokens = tokens; 16 } 17 nestedBw.setPropertyValue(tokens, pv); 18 } 19     else { 20 setPropertyValue(tokens, pv); 21 } 22 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



找一个合适的BeanWrapper，这里就是自身，然后执行17行的setPropertyValue方法进入最后一步，方法非常长，截取核心的一段：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 final Method writeMethod = (pd instanceof GenericTypeAwarePropertyDescriptor ?
 2     ((GenericTypeAwarePropertyDescriptor) pd).getWriteMethodForActualAccess() :
 3     pd.getWriteMethod());
 4     if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers()) && !writeMethod.isAccessible()) {
 5     if (System.getSecurityManager()!= null) {
 6         AccessController.doPrivileged(new PrivilegedAction<Object>() {
 7                 public Object run() { 8                     writeMethod.setAccessible(true);
 9                     return null; 10 } 11 }); 12 } 13         else { 14             writeMethod.setAccessible(true); 15 } 16 } 17     final Object value = valueToApply; 18     if (System.getSecurityManager() != null) { 19     try { 20         AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() { 21             public Object run() throws Exception { 22 writeMethod.invoke(object, value); 23                 return null; 24 } 25 }, acc); 26 } 27     catch (PrivilegedActionException ex) { 28         throw ex.getException(); 29 } 30 } 31 else { 32     writeMethod.invoke(this.object, value); 33 }                </pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



大致流程就是两步：

（1）拿到写方法并将方法的可见性设置为true

（2）拿到Value值，对Bean通过反射调用写方法

这样完成了对于Bean属性值的设置。

## Aware注入

接下来是Aware注入。在使用Spring的时候我们将自己的Bean实现BeanNameAware接口、BeanFactoryAware接口等，依赖容器帮我们注入当前Bean的名称或者Bean工厂，其代码实现先追溯到上面doCreateBean方法的42行initializeBean方法：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) { 2     if (System.getSecurityManager() != null) {
 3         AccessController.doPrivileged(new PrivilegedAction<Object>() {
 4             public Object run() { 5                 invokeAwareMethods(beanName, bean);
 6                 return null;
 7             }
 8         }, getAccessControlContext());
 9 } 10     else { 11 invokeAwareMethods(beanName, bean); 12 } 13         
14     Object wrappedBean = bean; 15     if (mbd == null || !mbd.isSynthetic()) { 16         wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName); 17 } 18 
19     try { 20 invokeInitMethods(beanName, wrappedBean, mbd); 21 } 22     catch (Throwable ex) { 23         throw new BeanCreationException( 24                 (mbd != null ? mbd.getResourceDescription() : null), 25                 beanName, "Invocation of init method failed", ex); 26 } 27 
28     if (mbd == null || !mbd.isSynthetic()) { 29         wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName); 30 } 31     return wrappedBean; 32 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



看一下上面第5行的实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 private void invokeAwareMethods(final String beanName, final Object bean) { 2     if (bean instanceof BeanNameAware) { 3         ((BeanNameAware) bean).setBeanName(beanName);
 4     }
 5     if (bean instanceof BeanClassLoaderAware) { 6         ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
 7     }
 8     if (bean instanceof BeanFactoryAware) { 9         ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this); 10 } 11 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



看到这里判断，如果bean是BeanNameAware接口的实现类会调用setBeanName方法、如果bean是BeanClassLoaderAware接口的实现类会调用setBeanClassLoader方法、如果是BeanFactoryAware接口的实现类会调用setBeanFactory方法，注入对应的属性值。

调用BeanPostProcessor的postProcessBeforeInitialization方法

上面initializeBean方法再看16行其实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) 2         throws BeansException { 3 
 4     Object result = existingBean; 5     for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) { 6         result = beanProcessor.postProcessBeforeInitialization(result, beanName); 7         if (result == null) {
 8             return result; 9 } 10 } 11     return result; 12 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



遍历每个BeanPostProcessor接口实现，调用postProcessBeforeInitialization方法，这个接口的调用时机之后会总结，这里就代码先简单提一下。

## 调用初始化方法

initializeBean方法的20行，调用Bean的初始化方法，看一下实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd) 2         throws Throwable { 3 
 4     boolean isInitializingBean = (bean instanceof InitializingBean); 5     if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
 6         if (logger.isDebugEnabled()) { 7             logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
 8         }
 9         if (System.getSecurityManager() != null) { 10             try { 11                 AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() { 12                     public Object run() throws Exception { 13 ((InitializingBean) bean).afterPropertiesSet(); 14                         return null; 15 } 16 }, getAccessControlContext()); 17 } 18             catch (PrivilegedActionException pae) { 19                 throw pae.getException(); 20 } 21 } 22         else { 23 ((InitializingBean) bean).afterPropertiesSet(); 24 } 25 } 26 
27     if (mbd != null) { 28         String initMethodName = mbd.getInitMethodName(); 29         if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
30                     !mbd.isExternallyManagedInitMethod(initMethodName)) { 31 invokeCustomInitMethod(beanName, bean, mbd); 32 } 33 } 34 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



看到，代码做了两件事情：

1、先判断Bean是否InitializingBean的实现类，是的话，将Bean强转为InitializingBean，直接调用afterPropertiesSet()方法

2、尝试去拿init-method，假如有的话，通过反射，调用initMethod

因此，两种方法各有优劣：使用实现InitializingBean接口的方式效率更高一点，因为init-method方法是通过反射进行调用的；从另外一个角度讲，使用init-method方法之后和Spring的耦合度会更低一点。具体使用哪种方式调用初始化方法，看个人喜好。

调用BeanPostProcessor的postProcessAfterInitialization方法

最后一步，initializeBean方法的29行：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) 2         throws BeansException { 3 
 4     Object result = existingBean; 5     for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) { 6         result = beanProcessor.postProcessAfterInitialization(result, beanName); 7         if (result == null) {
 8             return result; 9 } 10 } 11     return result; 12 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



同样遍历BeanPostProcessor，调用postProcessAfterInitialization方法。因此对于BeanPostProcessor方法总结一下：

1、在初始化每一个Bean的时候都会调用每一个配置的BeanPostProcessor的方法

2、在Bean属性设置、Aware设置后调用postProcessBeforeInitialization方法

3、在初始化方法调用后调用postProcessAfterInitialization方法

注册需要执行销毁方法的Bean

接下来看一下最上面doCreateBean方法的第83行registerDisposableBeanIfNecessary(beanName, bean, mbd)这一句，完成了创建Bean的最后一件事情：注册需要执行销毁方法的Bean。

看一下方法的实现：



![复制代码](http://common.cnblogs.com/images/copycode.gif)

<pre> 1 protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) { 2     AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
 3     if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) { 4         if (mbd.isSingleton()) { 5             // Register a DisposableBean implementation that performs all destruction 6             // work for the given bean: DestructionAwareBeanPostProcessors, 7             // DisposableBean interface, custom destroy method.
 8             registerDisposableBean(beanName,
 9                     new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc)); 10 } 11         else { 12             // A bean with a custom scope...
13             Scope scope = this.scopes.get(mbd.getScope()); 14             if (scope == null) { 15                 throw new IllegalStateException("No Scope registered for scope '" + mbd.getScope() + "'"); 16 } 17 scope.registerDestructionCallback(beanName, 18                     new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc)); 19 } 20 } 21 }</pre>

![复制代码](http://common.cnblogs.com/images/copycode.gif)



其中第3行第一个判断为必须不是prototype（原型）的，第二个判断requiresDestruction方法的实现为：



<pre>1 protected boolean requiresDestruction(Object bean, RootBeanDefinition mbd) { 2     return (bean != null &&
3             (bean instanceof DisposableBean || mbd.getDestroyMethodName() != null ||
4 hasDestructionAwareBeanPostProcessors())); 5 }</pre>



要注册销毁方法，Bean需要至少满足以下三个条件之一：

（1）Bean是DisposableBean的实现类，此时执行DisposableBean的接口方法destroy()

（2）Bean标签中有配置destroy-method属性，此时执行destroy-method配置指定的方法

（3）当前Bean对应的BeanFactory中持有DestructionAwareBeanPostProcessor接口的实现类，此时执行DestructionAwareBeanPostProcessor的接口方法postProcessBeforeDestruction

在满足上面三个条件之一的情况下，容器便会注册销毁该Bean，注册Bean的方法很简单，见registerDisposableBean方法实现：



<pre>1 public void registerDisposableBean(String beanName, DisposableBean bean) { 2     synchronized (this.disposableBeans) { 3         this.disposableBeans.put(beanName, bean); 4 } 5 }</pre>



容器销毁的时候，会遍历disposableBeans，逐一执行销毁方法。

## 流程总结

本文和上篇文章分析了Spring Bean初始化的步骤，最后用一幅图总结一下Spring Bean初始化的流程：

![](https://images2015.cnblogs.com/blog/801753/201702/801753-20170204111521089-1301937796.png)

图只是起梳理流程作用，抛砖引玉，具体代码实现还需要网友朋友们照着代码自己去一步一步分析。

