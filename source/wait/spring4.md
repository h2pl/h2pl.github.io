---
title: Spring源码剖析4：其余方式获取Bean的过程分析
date: 2018-06-03 22:27:54
tags:
    - Spring
categories:
	- 后端
	- Spring
---

本系列文章首发于我的个人博客：https://h2pl.github.io/

欢迎阅览我的CSDN专栏：Spring源码解析 https://blog.csdn.net/column/details/21851.html

部分代码会放在我的的Github：https://github.com/h2pl/

<!-- more -->

## 原型Bean加载过程

之前的文章，分析了非懒加载的单例Bean整个加载过程，除了非懒加载的单例Bean之外，Spring中还有一种Bean就是原型（Prototype）的Bean，看一下定义方式：




<pre>1 <?xml version="1.0" encoding="UTF-8"?>
2 <beans xmlns="http://www.springframework.org/schema/beans"
3     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
4     xsi:schemaLocation="http://www.springframework.org/schema/beans 5     http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
6 
7     <bean id="prototypeBean" class="org.xrq.action.PrototypeBean" scope="prototype"  />
8     
9 </beans></pre>




原型Bean加载流程总得来说和单例Bean差不多，看一下不同之处，在AbstractBeanFactory的doGetBean的方法的这一步：




<pre> 1 else if (mbd.isPrototype()) { 2     // It's a prototype -> create a new instance.
 3     Object prototypeInstance = null;
 4     try { 5         beforePrototypeCreation(beanName);
 6         prototypeInstance = createBean(beanName, mbd, args); 7     }
 8     finally { 9 afterPrototypeCreation(beanName); 10 } 11     bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd); 12 }</pre>




第6行createBean是一样的，原型Bean实例化的主要区别就在于第6行，它是直接创建bean的，而单例bean我们再对比一下：




<pre> 1 if (mbd.isSingleton()) { 2     sharedInstance = getSingleton(beanName, new ObjectFactory() { 3         public Object getObject() throws BeansException { 4             try { 5                 return createBean(beanName, mbd, args); 6             }
 7             catch (BeansException ex) { 8                 // Explicitly remove instance from singleton cache: It might have been put there 9                 // eagerly by the creation process, to allow for circular reference resolution. 10                 // Also remove any beans that received a temporary reference to the bean.
11 destroySingleton(beanName); 12                 throw ex; 13 } 14 } 15 }); 16     bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd); 17 }</pre>




它优先会尝试getSington，即先尝试从singletonObjects中获取一下bean是否存在，如果存在直接返回singletonObjects中的bean对象。

接着，我们看到原型bean创建和单例bean创建的区别还在于第5行和第9行，先看第5行的代码：




<pre> 1 protected void beforePrototypeCreation(String beanName) { 2     Object curVal = this.prototypesCurrentlyInCreation.get();
 3     if (curVal == null) {
 4         this.prototypesCurrentlyInCreation.set(beanName);
 5     }
 6     else if (curVal instanceof String) { 7         Set<String> beanNameSet = new HashSet<String>(2);
 8         beanNameSet.add((String) curVal);
 9 beanNameSet.add(beanName); 10         this.prototypesCurrentlyInCreation.set(beanNameSet); 11 } 12     else { 13         Set<String> beanNameSet = (Set<String>) curVal; 14 beanNameSet.add(beanName); 15 } 16 }</pre>




这段主要是说bean在创建前要把当前beanName设置到ThreadLocal中去，其目的是保证多线程不会同时创建同一个bean。接着看第9行的代码实现，即bean创建之后做了什么：




<pre> 1 protected void afterPrototypeCreation(String beanName) { 2     Object curVal = this.prototypesCurrentlyInCreation.get();
 3     if (curVal instanceof String) { 4         this.prototypesCurrentlyInCreation.remove();
 5     }
 6     else if (curVal instanceof Set) { 7         Set<String> beanNameSet = (Set<String>) curVal;
 8         beanNameSet.remove(beanName);
 9         if (beanNameSet.isEmpty()) { 10             this.prototypesCurrentlyInCreation.remove(); 11 } 12 } 13 }</pre>




很好理解，就是把当前bean移除一下，这样其它线程就可以创建bean了。第11行的代码不看了，意思是如果bean是FactoryBean的实现类的话，调用getObject()方法获取真正的对象。

 

## byName源码实现

Spring有为开发者提供Autowire（自动装配）的功能，自动装配最常用的就是byName和byType这两种属性。由于自动装配是为了解决对象注入导致的<property>过多的问题，因此很容易找到byName与byType的Spring源码实现应该在属性注入这一块，定位到属性注入的代码AbstractAutowireCapableBeanFactory的populateBean方法，直接截取重点：




<pre> 1 if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
 2         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) { 3     MutablePropertyValues newPvs = new MutablePropertyValues(pvs); 4 
 5     // Add property values based on autowire by name if applicable.
 6     if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) { 7         autowireByName(beanName, mbd, bw, newPvs);
 8     }
 9 
10     // Add property values based on autowire by type if applicable.
11     if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) { 12 autowireByType(beanName, mbd, bw, newPvs); 13 } 14 
15     pvs = newPvs; 16 }</pre>




看到第6行~第8行判断是否byName形式，是就执行byName自动装配代码；第11行~第13行判断是否byType形式，是就执行byType自动装配代码。那么首先看一下第7行的byName代码实现：




<pre> 1 protected void autowireByName( 2         String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
 3 
 4     String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw); 5     for (String propertyName : propertyNames) { 6         if (containsBean(propertyName)) { 7             Object bean = getBean(propertyName); 8             pvs.add(propertyName, bean);
 9 registerDependentBean(propertyName, beanName); 10             if (logger.isDebugEnabled()) { 11                 logger.debug("Added autowiring by name from bean name '" + beanName +
12                         "' via property '" + propertyName + "' to bean named '" + propertyName + "'"); 13 } 14 } 15         else { 16             if (logger.isTraceEnabled()) { 17                 logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
18                         "' by name: no matching bean found"); 19 } 20 } 21 } 22 }</pre>




篇幅问题，代码不一层层跟了，逻辑梳理一下：

*   第4行，找到Bean中不是简单属性的属性，这句话有点绕，意思就是找到属性是对象类型的属性，但也不是所有的对象类型都会被找到，比如CharSequence类型、Number类型、Date类型、URL类型、URI类型、Locale类型、Class类型就会忽略，具体可见BeanUtils的isSimpleProperty方法
*   第5行~第7行，遍历所有被找到的属性，如果bean定义中包含了属性名，那么先实例化该属性名对应的bean
*   第9行registerDependentBean，注册一下当前bean的依赖bean，用于在某个bean被销毁前先将其依赖的bean销毁

其余代码都是一些打日志的，没什么好说的。

## byType源码实现

上面说了byName的源码实现，接下来看一下byType源码实现：




<pre> 1 protected void autowireByType( 2         String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
 3 
 4     TypeConverter converter = getCustomTypeConverter(); 5     if (converter == null) {
 6         converter = bw; 7     }
 8 
 9     Set<String> autowiredBeanNames = new LinkedHashSet<String>(4); 10     String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw); 11     for (String propertyName : propertyNames) { 12         try { 13             PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName); 14             // Don't try autowiring by type for type Object: never makes sense, 15             // even if it technically is a unsatisfied, non-simple property.
16             if (!Object.class.equals(pd.getPropertyType())) { 17                 MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd); 18                 // Do not allow eager init for type matching in case of a prioritized post-processor.
19                 boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass()); 20                 DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager); 21                 Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter); 22                 if (autowiredArgument != null) { 23 pvs.add(propertyName, autowiredArgument); 24 } 25                 for (String autowiredBeanName : autowiredBeanNames) { 26 registerDependentBean(autowiredBeanName, beanName); 27                     if (logger.isDebugEnabled()) { 28                         logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
29                                 propertyName + "' to bean named '" + autowiredBeanName + "'"); 30 } 31 } 32 autowiredBeanNames.clear(); 33 } 34 } 35         catch (BeansException ex) { 36             throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex); 37 } 38 } 39 }</pre>




前面一样，到第10行都是找到Bean中属性是对象类型的属性。

接着就是遍历一下PropertyName，获取PropertyName对应的属性描述，注意一下16行的判断及其对应的注释：不要尝试自动装配Object类型，这没有任何意义，即使从技术角度看它是一个非简单的对象属性。

第18行~第20行跳过（没有太明白是干什么的），byType实现的源码主要在第21行的方法resolveDependency中，这个方法是AbstractAutowireCapableBeanFactory类的实现类DefaultListableBeanFactory中的方法：




<pre> 1 public Object resolveDependency(DependencyDescriptor descriptor, String beanName, 2     Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException  { 3 
 4     descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
 5     if (descriptor.getDependencyType().equals(ObjectFactory.class)) {
 6         return new DependencyObjectFactory(descriptor, beanName); 7     }
 8     else if (descriptor.getDependencyType().equals(javaxInjectProviderClass)) { 9         return new DependencyProviderFactory().createDependencyProvider(descriptor, beanName); 10 } 11     else { 12         return doResolveDependency(descriptor, descriptor.getDependencyType(), beanName, autowiredBeanNames, typeConverter); 13 } 14 }</pre>




这里判断一下要自动装配的属性是ObjectFactory.class还是javaxInjectProviderClass还是其他的，我们装配的是其他的，看一下12行的代码实现：




<pre> 1 protected Object doResolveDependency(DependencyDescriptor descriptor, Class<?> type, String beanName, 2     Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException  { 3 
 4     Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor); 5     if (value != null) {
 6         if (value instanceof String) { 7             String strVal = resolveEmbeddedValue((String) value); 8             BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
 9             value = evaluateBeanDefinitionString(strVal, bd); 10 } 11         TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter()); 12         return converter.convertIfNecessary(value, type); 13 } 14 
15     if (type.isArray()) { 16 ... 17 } 18     else if (Collection.class.isAssignableFrom(type) && type.isInterface()) { 19 ... 20 } 21     else if (Map.class.isAssignableFrom(type) && type.isInterface()) { 22 ... 23 } 24     else { 25         Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor); 26         if (matchingBeans.isEmpty()) { 27             if (descriptor.isRequired()) { 28                 raiseNoSuchBeanDefinitionException(type, "", descriptor); 29 } 30             return null; 31 } 32         if (matchingBeans.size() > 1) { 33             String primaryBeanName = determinePrimaryCandidate(matchingBeans, descriptor); 34             if (primaryBeanName == null) { 35                 throw new NoSuchBeanDefinitionException(type, "expected single matching bean but found " +
36                         matchingBeans.size() + ": " + matchingBeans.keySet()); 37 } 38             if (autowiredBeanNames != null) { 39 autowiredBeanNames.add(primaryBeanName); 40 } 41             return matchingBeans.get(primaryBeanName); 42 } 43         // We have exactly one match.
44         Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next(); 45         if (autowiredBeanNames != null) { 46 autowiredBeanNames.add(entry.getKey()); 47 } 48         return entry.getValue(); 49 } 50 }</pre>




第四行结果是null不看了，为了简化代码Array装配、Collection装配、Map装配的代码都略去了，重点看一下普通属性的装配。首先是第25行获取一下自动装配的候选者：




<pre> 1 protected Map<String, Object> findAutowireCandidates( 2     String beanName, Class requiredType, DependencyDescriptor descriptor) {
 3 
 4     String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors( 5             this, requiredType, true, descriptor.isEager());
 6     Map<String, Object> result = new LinkedHashMap<String, Object>(candidateNames.length);
 7     for (Class autowiringType : this.resolvableDependencies.keySet()) {
 8         if (autowiringType.isAssignableFrom(requiredType)) { 9             Object autowiringValue = this.resolvableDependencies.get(autowiringType); 10             autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType); 11             if (requiredType.isInstance(autowiringValue)) { 12 result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue); 13                 break; 14 } 15 } 16 } 17     for (String candidateName : candidateNames) { 18         if (!candidateName.equals(beanName) && isAutowireCandidate(candidateName, descriptor)) { 19 result.put(candidateName, getBean(candidateName)); 20 } 21 } 22     return result; 23 }</pre>




代码逻辑整理一下：

*   首先获取候选者bean名称，通过DefaultListableBeanFactory的getBeanNamesForType方法，即找一下所有的Bean定义中指定Type的实现类或者子类
*   接着第7行~第16行的判断要自动装配的类型是不是要自动装配的纠正类型，这个在[【Spring源码分析】非懒加载的单例Bean初始化前后的一些操作](http://www.cnblogs.com/xrq730/p/6670457.html)一文讲PrepareBeanFactory方法的时候有讲过，如果要自动装配的类型是纠正类型，比如是一个ResourceLoader，那么就会为该类型生成一个代理实例，具体可以看一下第10行的AutowireUtils.resolveAutowiringValue方法的实现  
*   正常来说都是执行的第17行~第21行的代码，逐个判断查找一下beanName对应的BeanDefinition，判断一下是不是自动装配候选者，默认都是的，如果<bean>的autowire-candidate属性设置为false就不是

这样，拿到所有待装配对象的实现类或者子类的候选者，组成一个Map，Key为beanName，Value为具体的Bean。接着回看获取Bean之后的逻辑：




<pre> 1 Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor); 2     if (matchingBeans.isEmpty()) { 3         if (descriptor.isRequired()) { 4             raiseNoSuchBeanDefinitionException(type, "", descriptor);
 5         }
 6         return null;
 7     }
 8     if (matchingBeans.size() > 1) {
 9         String primaryBeanName = determinePrimaryCandidate(matchingBeans, descriptor); 10         if (primaryBeanName == null) { 11             throw new NoSuchBeanDefinitionException(type, "expected single matching bean but found " +
12                     matchingBeans.size() + ": " + matchingBeans.keySet()); 13 } 14         if (autowiredBeanNames != null) { 15 autowiredBeanNames.add(primaryBeanName); 16 } 17         return matchingBeans.get(primaryBeanName); 18 } 19     // We have exactly one match.
20     Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next(); 21     if (autowiredBeanNames != null) { 22 autowiredBeanNames.add(entry.getKey()); 23 } 24     ... 25 }</pre>




整理一下逻辑：

*   如果拿到的Map是空的且属性必须注入，抛异常
*   如果拿到的Map中有多个候选对象，判断其中是否有<bean>中属性配置为"primary=true"的，有就拿执行第13行~第15行的代码，没有就第8行的方法返回null，抛异常，这个异常的描述相信Spring用的比较多的应该比较熟悉
*   如果拿到的Map中只有一个候选对象，直接拿到那个 

通过这样一整个流程，实现了byType自动装配，byType自动装配流程比较长，中间细节比较多，还需要多看看才能弄明白。

最后注意一点，即所有待注入的PropertyName-->PropertyValue映射拿到之后都只是放在MutablePropertyValues中，最后由AbstractPropertyAccessor类的setPropertyValues方法遍历并进行逐一注入。

## 通过FactoryBean获取Bean实例源码实现

我们知道可以通过实现FactoryBean接口，重写getObject()方法实现个性化定制Bean的过程，这部分我们就来看一下Spring源码是如何实现通过FactoryBean获取Bean实例的。代码直接定位到AbstractBeanFactory的doGetBean方法创建单例Bean这部分：




<pre> 1 // Create bean instance.
 2 if (mbd.isSingleton()) { 3     sharedInstance = getSingleton(beanName, new ObjectFactory() { 4         public Object getObject() throws BeansException { 5             try { 6                 return createBean(beanName, mbd, args); 7             }
 8             catch (BeansException ex) { 9                 // Explicitly remove instance from singleton cache: It might have been put there 10                 // eagerly by the creation process, to allow for circular reference resolution. 11                 // Also remove any beans that received a temporary reference to the bean.
12 destroySingleton(beanName); 13                 throw ex; 14 } 15 } 16 }); 17     bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd); 18 }</pre>




FactoryBean首先是个Bean且被实例化出来成为一个对象之后才能调用getObject()方法，因此还是会执行第3行~第16行的代码，这段代码之前分析过了就不说了。之后执行第17行的方法：




<pre> 1 protected Object getObjectForBeanInstance( 2         Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {
 3 
 4     // Don't let calling code try to dereference the factory if the bean isn't a factory.
 5     if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) { 6         throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass()); 7     }
 8 
 9     // Now we have the bean instance, which may be a normal bean or a FactoryBean. 10     // If it's a FactoryBean, we use it to create a bean instance, unless the 11     // caller actually wants a reference to the factory.
12     if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) { 13         return beanInstance; 14 } 15 
16     Object object = null; 17     if (mbd == null) { 18         object = getCachedObjectForFactoryBean(beanName); 19 } 20     if (object == null) { 21         // Return bean instance from factory.
22         FactoryBean factory = (FactoryBean) beanInstance; 23         // Caches object obtained from FactoryBean if it is a singleton.
24         if (mbd == null && containsBeanDefinition(beanName)) { 25             mbd = getMergedLocalBeanDefinition(beanName); 26 } 27         boolean synthetic = (mbd != null && mbd.isSynthetic()); 28         object = getObjectFromFactoryBean(factory, beanName, !synthetic); 29 } 30     return object; 31 }</pre>




首先第5行~第7行判断一下是否beanName以"&"开头并且不是FactoryBean的实现类，不满足则抛异常，因为beanName以"&"开头是FactoryBean的实现类bean定义的一个特征。

接着判断第12行~第14行，如果：

*   bean不是FactoryBean的实现类
*   beanName以"&"开头

这两种情况，都直接把生成的bean对象返回出去，不会执行余下的流程。

最后流程走到第16行~第30行，最终调用getObject()方法实现个性化定制bean，先执行第28行的方法：




<pre> 1 protected Object getObjectFromFactoryBean(FactoryBean factory, String beanName, boolean shouldPostProcess) { 2     if (factory.isSingleton() && containsSingleton(beanName)) { 3         synchronized (getSingletonMutex()) { 4             Object object = this.factoryBeanObjectCache.get(beanName);
 5             if (object == null) {
 6                 object = doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess); 7                 this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT)); 8             }
 9             return (object != NULL_OBJECT ? object : null); 10 } 11 } 12     else { 13         return doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess); 14 } 15 }</pre>




第1行~第11行的代码与第12行~第13行的代码最终都是一样的，调用了如下一段：




<pre> 1 private Object doGetObjectFromFactoryBean( 2         final FactoryBean factory, final String beanName, final boolean shouldPostProcess) 3         throws BeanCreationException { 4 
 5     Object object;
 6     try { 7         if (System.getSecurityManager() != null) {
 8             AccessControlContext acc = getAccessControlContext(); 9             try { 10                 object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() { 11                     public Object run() throws Exception { 12                             return factory.getObject(); 13 } 14 }, acc); 15 } 16             catch (PrivilegedActionException pae) { 17                 throw pae.getException(); 18 } 19 } 20         else { 21             object = factory.getObject(); 22 } 23 } 24     catch (FactoryBeanNotInitializedException ex) { 25         throw new BeanCurrentlyInCreationException(beanName, ex.toString()); 26 } 27     catch (Throwable ex) { 28         throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex); 29 } 30         
31     // Do not accept a null value for a FactoryBean that's not fully 32     // initialized yet: Many FactoryBeans just return null then.
33     if (object == null && isSingletonCurrentlyInCreation(beanName)) { 34         throw new BeanCurrentlyInCreationException( 35                 beanName, "FactoryBean which is currently in creation returned null from getObject"); 36 } 37 
38     if (object != null && shouldPostProcess) { 39         try { 40             object = postProcessObjectFromFactoryBean(object, beanName); 41 } 42         catch (Throwable ex) { 43             throw new BeanCreationException(beanName, "Post-processing of the FactoryBean's object failed", ex); 44 } 45 } 46 
47     return object; 48 }</pre>




第12行和第21行的代码，都一样，最终调用getObject()方法获取对象。回过头去看之前的getObjectFromFactoryBean方法，虽然if...else...逻辑最终都是调用了以上的方法，但是区别在于：

*   如果FactoryBean接口实现类的isSington方法返回的是true，那么每次调用getObject方法的时候会优先尝试从FactoryBean对象缓存中取目标对象，有就直接拿，没有就创建并放入FactoryBean对象缓存，这样保证了每次单例的FactoryBean调用getObject()方法后最终拿到的目标对象一定是单例的，即在内存中都是同一份
*   如果FactoryBean接口实现类的isSington方法返回的是false，那么每次调用getObject方法的时候都会新创建一个目标对象