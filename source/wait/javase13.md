---
title: Java基础13：反射详解
date: 2018-05-01 22:26:21
tags:
    - Java反射
categories:
	- 后端
	- Java基础
---
本节主要介绍Java反射的原理，使用方法以及相关的技术细节，并且介绍了关于Class类，注解等内容。

具体代码在我的GitHub中可以找到
> https://github.com/h2pl/MyTech

文章首发于我的个人博客：
> https://h2pl.github.io/2018/05/01/javase13

更多关于Java后端学习的内容请到我的CSDN博客上查看：
> https://blog.csdn.net/a724888

<!-- more -->

## 回顾：什么是反射？

反射(Reflection)是Java 程序开发语言的特征之一，它允许运行中的 Java 程序获取自身的信息，并且可以操作类或对象的内部属性。
Oracle官方对反射的解释是

> Reflection enables Java code to discover information about the fields, methods and constructors of loaded classes, and to use reflected fields, methods, and constructors to operate on their underlying counterparts, within security restrictions.

> The API accommodates applications that need access to either the public members of a target object (based on its runtime class) or the members declared by a given class. It also allows programs to suppress default reflective access control.
> 
>  简而言之，通过反射，我们可以在运行时获得程序或程序集中每一个类型的成员和成员的信息。
>
> 程序中一般的对象的类型都是在编译期就确定下来的，而Java反射机制可以动态地创建对象并调用其属性，这样的对象的类型在编译期是未知的。所以我们可以通过反射机制直接创建对象，即使这个对象的类型在编译期是未知的。
> 
>  反射的核心是JVM在运行时才动态加载类或调用方法/访问属性，它不需要事先（写代码的时候或编译期）知道运行对象是谁。

Java反射框架主要提供以下功能：

> 1.在运行时判断任意一个对象所属的类；

> 2.在运行时构造任意一个类的对象；

> 3.在运行时判断任意一个类所具有的成员变量和方法（通过反射甚至可以调用private方法）；

> 4.在运行时调用任意一个对象的方法

> 重点：是运行时而不是编译时


## 反射的主要用途

>  很多人都认为反射在实际的Java开发应用中并不广泛，其实不然。

>  当我们在使用IDE(如Eclipse，IDEA)时，当我们输入一个对象或类并想调用它的属性或方法时，一按点号，编译器就会自动列出它的属性或方法，这里就会用到反射。

>  反射最重要的用途就是开发各种通用框架。

>  很多框架（比如Spring）都是配置化的（比如通过XML文件配置JavaBean,Action之类的），为了保证框架的通用性，它们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须用到反射——运行时动态加载需要加载的对象。

>  举一个例子，在运用Struts 2框架的开发中我们一般会在struts.xml里去配置Action，比如：

    <action name="login"
                   class="org.ScZyhSoft.test.action.SimpleLoginAction"
                   method="execute">
               <result>/shop/shop-index.jsp</result>
               <result name="error">login.jsp</result>
           </action>
配置文件与Action建立了一种映射关系，当View层发出请求时，请求会被StrutsPrepareAndExecuteFilter拦截，然后StrutsPrepareAndExecuteFilter会去动态地创建Action实例。

——比如我们请求login.action，那么StrutsPrepareAndExecuteFilter就会去解析struts.xml文件，检索action中name为login的Action，并根据class属性创建SimpleLoginAction实例，并用invoke方法来调用execute方法，这个过程离不开反射。

> 对与框架开发人员来说，反射虽小但作用非常大，它是各种容器实现的核心。而对于一般的开发者来说，不深入框架开发则用反射用的就会少一点，不过了解一下框架的底层机制有助于丰富自己的编程思想，也是很有益的。

## 反射的基础：关于Class类

更多关于Class类和Object类的原理和介绍请见上一节

> 1、Class是一个类，一个描述类的类（也就是描述类本身），封装了描述方法的Method，描述字段的Filed，描述构造器的Constructor等属性
> 
> 2、对象照镜子后（反射）可以得到的信息：某个类的数据成员名、方法和构造器、某个类到底实现了哪些接口。
> 
> 3、对于每个类而言，JRE 都为其保留一个不变的 Class 类型的对象。一个Class对象包含了特定某个类的有关信息。
>     
> 4、Class 对象只能由系统建立对象
> 
> 5、一个类在 JVM 中只会有一个Class实例

    //总结一下就是，JDK有一个类叫做Class，这个类用来封装所有Java类型，包括这些类的所有信息，JVM中类信息是放在方法区的。
    
    //所有类在加载后，JVM会为其在堆中创建一个Class<类名称>的对象，并且每个类只会有一个Class对象，这个类的所有对象都要通过Class<类名称>来进行实例化。
    
    //上面说的是JVM进行实例化的原理，当然实际上在Java写代码时只需要用 类名称就可以进行实例化了。
    
    public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
    虚拟机会保持唯一一
                //通过类名.class获得唯一的Class对象。
                Class<UserBean> cls = UserBean.class;
                //通过integer.TYPEl来获取Class对象
                Class<Integer> inti = Integer.TYPE;
              //接口本质也是一个类，一样可以通过.class获取
                Class<User> userClass = User.class;

## 反射的基本运用

上面我们提到了反射可以用于判断任意对象所属的类，获得Class对象，构造任意一个对象以及调用一个对象。这里我们介绍一下基本反射功能的实现(反射相关的类一般都在java.lang.relfect包里)。

1、获得Class对象方法有三种

(1)使用Class类的forName静态方法:
    
     public static Class<?> forName(String className)
    ``` 
    在JDBC开发中常用此方法加载数据库驱动:
    要使用全类名来加载这个类，一般数据库驱动的配置信息会写在配置文件中。加载这个驱动前要先导入jar包
    ```java
     Class.forName(driver);
(2)直接获取某一个对象的class，比如:

    //Class<?>是一个泛型表示，用于获取一个类的类型。
    Class<?> klass = int.class;
    Class<?> classInt = Integer.TYPE;
(3)调用某个对象的getClass()方法,比如:
    
    StringBuilder str = new StringBuilder("123");
    Class<?> klass = str.getClass();
    
## 判断是否为某个类的实例

一般地，我们用instanceof关键字来判断是否为某个类的实例。同时我们也可以借助反射中Class对象的isInstance()方法来判断是否为某个类的实例，它是一个Native方法：

==public native boolean isInstance(Object obj);==

## 创建实例

通过反射来生成对象主要有两种方式。

（1）使用Class对象的newInstance()方法来创建Class对象对应类的实例。

注意：利用newInstance创建对象：调用的类必须有无参的构造器

    //Class<?>代表任何类的一个类对象。
    //使用这个类对象可以为其他类进行实例化
    //因为jvm加载类以后自动在堆区生成一个对应的*.Class对象
    //该对象用于让JVM对进行所有*对象实例化。
    Class<?> c = String.class;
    
    //Class<?> 中的 ? 是通配符，其实就是表示任意符合泛类定义条件的类，和直接使用 Class
    //效果基本一致，但是这样写更加规范，在某些类型转换时可以避免不必要的 unchecked 错误。
    
    Object str = c.newInstance();

（2）先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例。这种方法可以用指定的构造器构造类的实例。

    //获取String所对应的Class对象
    Class<?> c = String.class;
    //获取String类带一个String参数的构造器
    Constructor constructor = c.getConstructor(String.class);
    //根据构造器创建实例
    Object obj = constructor.newInstance("23333");
    System.out.println(obj);

## 获取方法
获取某个Class对象的方法集合，主要有以下几个方法：

getDeclaredMethods()方法返回类或接口声明的所有方法，==包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法==。

    public Method[] getDeclaredMethods() throws SecurityException

getMethods()方法返回某个类的所有公用（public）方法，==包括其继承类的公用方法。==
    
    public Method[] getMethods() throws SecurityException
getMethod方法返回一个特定的方法，其中第一个参数为方法名称，后面的参数为方法的参数对应Class的对象


    public Method getMethod(String name, Class<?>... parameterTypes)
只是这样描述的话可能难以理解，我们用例子来理解这三个方法：
本文中的例子用到了以下这些类，用于反射的测试。

    //注解类，可可用于表示方法，可以通过反射获取注解的内容。
        //Java注解的实现是很多注框架实现注解配置的基础
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Invoke {
    }

userbean的父类personbean

    public class PersonBean {
    private String name;

    int id;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    
    
}

接口user

    public interface User {
        public void login ();
    
    }

userBean实现user接口，继承personbean

    public class UserBean extends PersonBean implements User{
        @Override
        public void login() {
    
        }
    
        class B {
    
        }
    
        public String userName;
        protected int i;
        static int j;
        private int l;
        private long userId;
        public UserBean(String userName, long userId) {
            this.userName = userName;
            this.userId = userId;
        }
        public String getName() {
            return userName;
        }
        public long getId() {
            return userId;
        }
        @Invoke
        public static void staticMethod(String devName,int a) {
            System.out.printf("Hi %s, I'm a static method", devName);
        }
        @Invoke
        public void publicMethod() {
            System.out.println("I'm a public method");
        }
        @Invoke
        private void privateMethod() {
            System.out.println("I'm a private method");
        }
    }

1 getMethods和getDeclaredMethods的区别
    
    public class 动态加载类的反射 {
        public static void main(String[] args) {
            try {
                Class clazz = Class.forName("com.javase.反射.UserBean");
                for (Field field : clazz.getDeclaredFields()) {
    //                field.setAccessible(true);
                    System.out.println(field);
                }
                //getDeclaredMethod*()获取的是类自身声明的所有方法，包含public、protected和private方法。
                System.out.println("------共有方法------");
    //        getDeclaredMethod*()获取的是类自身声明的所有方法，包含public、protected和private方法。
    //            getMethod*()获取的是类的所有共有方法，这就包括自身的所有public方法，和从基类继承的、从接口实现的所有public方法。
                for (Method method : clazz.getMethods()) {
                    String name = method.getName();
                    System.out.println(name);
                    //打印出了UserBean.java的所有方法以及父类的方法
                }
                System.out.println("------独占方法------");
    
                for (Method method : clazz.getDeclaredMethods()) {
                    String name = method.getName();
                    System.out.println(name);
                }
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
    }
    


2 打印一个类的所有方法及详细信息：

    public class 打印所有方法 {
    
        public static void main(String[] args) {
            Class userBeanClass = UserBean.class;
            Field[] fields = userBeanClass.getDeclaredFields();
            //注意，打印方法时无法得到局部变量的名称，因为jvm只知道它的类型
            Method[] methods = userBeanClass.getDeclaredMethods();
            for (Method method : methods) {
                //依次获得方法的修饰符，返回类型和名称，外加方法中的参数
                String methodString = Modifier.toString(method.getModifiers()) + " " ; // private static
                methodString += method.getReturnType().getSimpleName() + " "; // void
                methodString += method.getName() + "("; // staticMethod
                Class[] parameters = method.getParameterTypes();
                Parameter[] p = method.getParameters();
    
                for (Class parameter : parameters) {
                    methodString += parameter.getSimpleName() + " " ; // String
                }
                methodString += ")";
                System.out.println(methodString);
            }
            //注意方法只能获取到其类型，拿不到变量名
    /*        public String getName()
            public long getId()
            public static void staticMethod(String int )
            public void publicMethod()
            private void privateMethod()*/
        }
    }
    
## 获取构造器信息

获取类构造器的用法与上述获取方法的用法类似。主要是通过Class类的getConstructor方法得到Constructor类的一个实例，而Constructor类有一个newInstance方法可以创建一个对象实例:

    public class 打印构造方法 {
        public static void main(String[] args) {
            // constructors
            Class<?> clazz = UserBean.class;
    
            Class userBeanClass = UserBean.class;
            //获得所有的构造方法
            Constructor[] constructors = userBeanClass.getDeclaredConstructors();
            for (Constructor constructor : constructors) {
                String s = Modifier.toString(constructor.getModifiers()) + " ";
                s += constructor.getName() + "(";
                //构造方法的参数类型
                Class[] parameters = constructor.getParameterTypes();
                for (Class parameter : parameters) {
                    s += parameter.getSimpleName() + ", ";
                }
                s += ")";
                System.out.println(s);
                //打印结果//public com.javase.反射.UserBean(String, long, )

            }
        }
    }

## 获取类的成员变量（字段）信息
主要是这几个方法，在此不再赘述：

getFiled: 访问公有的成员变量
getDeclaredField：所有已声明的成员变量。但不能得到其父类的成员变量
getFileds和getDeclaredFields用法同上（参照Method）

    public class 打印成员变量 {
        public static void main(String[] args) {
            Class userBeanClass = UserBean.class;
            //获得该类的所有成员变量，包括static private
            Field[] fields = userBeanClass.getDeclaredFields();
    
            for(Field field : fields) {
                //private属性即使不用下面这个语句也可以访问
    //            field.setAccessible(true);
    
                //因为类的私有域在反射中默认可访问，所以flag默认为true。
                String fieldString = "";
                fieldString += Modifier.toString(field.getModifiers()) + " "; // `private`
                fieldString += field.getType().getSimpleName() + " "; // `String`
                fieldString += field.getName(); // `userName`
                fieldString += ";";
                System.out.println(fieldString);
                
                //打印结果
    //            public String userName;
    //            protected int i;
    //            static int j;
    //            private int l;
    //            private long userId;
            }
    
        }
    }
    
## 调用方法
当我们从类中获取了一个方法后，我们就可以用invoke()方法来调用这个方法。invoke方法的原型为:

    public Object invoke(Object obj, Object... args)
            throws IllegalAccessException, IllegalArgumentException,
               InvocationTargetException
           
    public class 使用反射调用方法 {
        public static void main(String[] args) throws InvocationTargetException, IllegalAccessException, InstantiationException, NoSuchMethodException {
            Class userBeanClass = UserBean.class;
            //获取该类所有的方法，包括静态方法，实例方法。
            //此处也包括了私有方法，只不过私有方法在用invoke访问之前要设置访问权限
            //也就是使用setAccessible使方法可访问，否则会抛出异常
    //       // IllegalAccessException的解释是
    //        * An IllegalAccessException is thrown when an application tries
    // * to reflectively create an instance (other than an array),
    // * set or get a field, or invoke a method, but the currently
    // * executing method does not have access to the definition of
    // * the specified class, field, method or constructor.
    
    //        getDeclaredMethod*()获取的是类自身声明的所有方法，包含public、protected和private方法。
    //            getMethod*()获取的是类的所有共有方法，这就包括自身的所有public方法，和从基类继承的、从接口实现的所有public方法。
    
            //就是说，当这个类，域或者方法被设为私有访问，使用反射调用但是却没有权限时会抛出异常。
            Method[] methods = userBeanClass.getDeclaredMethods(); // 获取所有成员方法
            for (Method method : methods) {
                //反射可以获取方法上的注解，通过注解来进行判断
                if (method.isAnnotationPresent(Invoke.class)) { // 判断是否被 @Invoke 修饰
                    //判断方法的修饰符是是static
                    if (Modifier.isStatic(method.getModifiers())) { // 如果是 static 方法
                        //反射调用该方法
                        //类方法可以直接调用，不必先实例化
                        method.invoke(null, "wingjay",2); // 直接调用，并传入需要的参数 devName
                    } else {
                        //如果不是类方法，需要先获得一个实例再调用方法
                        //传入构造方法需要的变量类型
                        Class[] params = {String.class, long.class};
                        //获取该类指定类型的构造方法
                        //如果没有这种类型的方法会报错
                        Constructor constructor = userBeanClass.getDeclaredConstructor(params); // 获取参数格式为 String,long 的构造函数
                        //通过构造方法的实例来进行实例化
                        Object userBean = constructor.newInstance("wingjay", 11); // 利用构造函数进行实例化，得到 Object
                        if (Modifier.isPrivate(method.getModifiers())) {
                            method.setAccessible(true); // 如果是 private 的方法，需要获取其调用权限
    //                        Set the {@code accessible} flag for this object to
    //     * the indicated boolean value.  A value of {@code true} indicates that
    //     * the reflected object should suppress Java language access
    //     * checking when it is used.  A value of {@code false} indicates
    //                                * that the reflected object should enforce Java language access checks.
                            //通过该方法可以设置其可见或者不可见，不仅可以用于方法
                            //后面例子会介绍将其用于成员变量
                                                //打印结果
    //            I'm a public method
    // Hi wingjay, I'm a static methodI'm a private method
                        }
                        method.invoke(userBean); // 调用 method，无须参数
                        
                        

                    }
                }
            }
        }
    }

## 利用反射创建数组

数组在Java里是比较特殊的一种类型，它可以赋值给一个Object Reference。下面我们看一看利用反射创建数组的例子：

    public class 用反射创建数组 {
        public static void main(String[] args) {
            Class<?> cls = null;
            try {
                cls = Class.forName("java.lang.String");
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            Object array = Array.newInstance(cls,25);
            //往数组里添加内容
            Array.set(array,0,"hello");
            Array.set(array,1,"Java");
            Array.set(array,2,"fuck");
            Array.set(array,3,"Scala");
            Array.set(array,4,"Clojure");
            //获取某一项的内容
            System.out.println(Array.get(array,3));
            //Scala
        }
    
    }
    
其中的Array类为java.lang.reflect.Array类。我们通过Array.newInstance()创建数组对象，它的原型是:

    public static Object newInstance(Class<?> componentType, int length)
            throws NegativeArraySizeException {
            return newArray(componentType, length);
        }
而newArray()方法是一个Native方法，它在Hotspot JVM里的具体实现我们后边再研究，这里先把源码贴出来

    private static native Object newArray(Class<?> componentType, int length)
            throws NegativeArraySizeException;
            
            
## Java的注解

9、注解(Annotation)

Java提供的注解，实际上可以通过反射的方式得到注解的内容 

> •从 JDK5.0 开始,Java 增加了对元数据(MetaData)的支持,也就是Annotation(注释)

> •Annotation其实就是代码里的特殊标记,这些标记可以在编译,类加载, 运行时被读取,并执行相应的处理.通过使用Annotation,程序员可以在不改变原有逻辑的情况下,在源文件中嵌入一些补充信息.

> •Annotation 可以像修饰符一样被使用,可用于修饰包,类,构造器, 方法,成员变量, 参数,局部变量的声明,这些信息被保存在Annotation的 “name=value”对中.

> •Annotation能被用来为程序元素(类,方法,成员变量等)设置元数据

基本的 Annotation

    •使用 Annotation时要在其前面增加@符号,并把该Annotation 当成一个修饰符使用.用于修饰它支持的程序元素
    
    •三个基本的Annotation:
        –@Override:限定重写父类方法,该注释只能用于方法
        –@Deprecated:用于表示某个程序元素(类,方法等)已过时
        –@SuppressWarnings:抑制编译器警告.
         
自定义 Annotation

> •定义新的 Annotation类型使用@interface关键字
> 
> •Annotation 的成员变量在Annotation
> 
> 定义中以无参数方法的形式来声明.其方法名和返回值定义了该成员的名字和类型.
> 
> •可以在定义Annotation的成员变量时为其指定初始值,指定成员变量的初始值可使用default关键字
> 
> •没有成员定义的Annotation称为标记;包含成员变量的Annotation称为元数据Annotation

    @Retention(RetentionPolicy.RUNTIME) //运行时检验  
    @Target(value = {ElementType.METHOD})  //作用在方法上  
    public @interface AgeValidator {  
      
        int min();  
        int max();  
        
        
注解的获取方法
    
    /** 
     * 通过反射才能获取注解 
     */  
    @Test  
    public void testAnnotation() throws Exception {  
        //这样的方式不能使用注解  
        Person3 person3 = new Person3();  
        person3.setAge(10);*/  
      
        String className = "com.java.reflection.Person3";  
        Class clazz = Class.forName(className);  
        Object obj = clazz.newInstance();  
      
        Method method = clazz.getDeclaredMethod("setAge",Integer.class);  
        int val =40;  
      
        //获取注解  
        Annotation annotation = method.getAnnotation(AgeValidator.class);  
        if (annotation != null){  
            if (annotation instanceof AgeValidator){  
                AgeValidator ageValidator = (AgeValidator) annotation;  
      
                if (val< ageValidator.min() || val>ageValidator.max()){  
                    throw new RuntimeException("数值超出范围");  
                }  
            }  
        }  
      
        method.invoke(obj, val);  
        System.out.println(obj);  
    }  
    
提取 Annotation信息
    
    •JDK5.0 在 java.lang.reflect包下新增了 AnnotatedElement接口,该接口代表程序中可以接受注释的程序元素
    •当一个 Annotation类型被定义为运行时Annotation后,该注释才是运行时可见,当 class文件被载入时保存在 class文件中的 Annotation才会被虚拟机读取
    •程序可以调用AnnotationElement对象的如下方法来访问 Annotation信息
    –获取 Annotation实例：
    •getAnnotation(Class<T> annotationClass)
    •getDeclaredAnnotations()
    •getParameterAnnotations()
    JDK 的元Annotation
    
    •JDK 的元Annotation 用于修饰其他Annotation 定义
    •@Retention:只能用于修饰一个 Annotation定义,用于指定该 Annotation可以保留多长时间,@Rentention包含一个RetentionPolicy类型的成员变量,使用 @Rentention时必须为该 value成员变量指定值:
        –RetentionPolicy.CLASS:编译器将把注释记录在 class文件中.当运行 Java程序时,JVM 不会保留注释.这是默认值
        –RetentionPolicy.RUNTIME:编译器将把注释记录在class文件中. 当运行 Java 程序时, JVM 会保留注释. 程序可以通过反射获取该注释
        –RetentionPolicy.SOURCE:编译器直接丢弃这种策略的注释
    •@Target: 用于修饰Annotation 定义,用于指定被修饰的 Annotation能用于修饰哪些程序元素.@Target 也包含一个名为 value的成员变量.
    •@Documented:用于指定被该元 Annotation修饰的 Annotation类将被 javadoc工具提取成文档.
    •@Inherited:被它修饰的 Annotation将具有继承性.如果某个类使用了被@Inherited 修饰的Annotation, 则其子类将自动具有该注释