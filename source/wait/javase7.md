---
title: Java基础7：关于Java类和包的那些事
date: 2018-04-24 17:03:51
tags:
categories:
	- 后端
	- Java基础
---
本文主要介绍了Java外部类和包的一些基本知识

内部类与匿名内部类的文章将在后面发布。

具体代码在我的GitHub中可以找到

https://github.com/h2pl/MyTech

喜欢的话麻烦点下星哈

文章首发于我的个人博客：

https://h2pl.github.io/2018/04/24/javase7

更多关于Java后端学习的内容请到我的CSDN博客上查看：

https://blog.csdn.net/a724888
<!-- more -->

## *.Java文件


问题：一个".java"源文件中是否可以包括多个类（不是内部类）？有什么限制？

> 　　答案：可以有多个类，但只能有一个public的类，并且public的类名必须与文件名相一致。一个文件中可以只有非public类，如果只有一个非public类，此类可以跟文件名不同。
> 
 

为什么一个java源文件中只能有一个public类？

　　在java编程思想（第四版）一书中有这样3段话（6.4 类的访问权限）：

> 　　1.每个编译单元（文件）都只能有一个public类，这表示，每个编译单元都有单一的公共接口，用public类来表现。该接口可以按要求包含众多的支持包访问权限的类。如果在某个编译单元内有一个以上的public类，编译器就会给出错误信息。
> 
> 　　2.public类的名称必须完全与含有该编译单元的文件名相同，包含大小写。如果不匹配，同样将得到编译错误。
> 
> 　　3.虽然不是很常用，但编译单元内完全不带public类也是可能的。在这种情况下，可以随意对文件命名。

总结相关的几个问题：

1、一个”.java”源文件中是否可以包括多个类（不是内部类）？有什么限制？

>   答：可以有多个类，但只能有一个public的类，并且public的类名必须与文件名相一致。

2、为什么一个文件中只能有一个public的类

>   答：编译器在编译时，针对一个java源代码文件（也称为“编译单元”）只会接受一个public类。否则报错。

3、在java文件中是否可以没有public类

>   答：public类不是必须的，java文件中可以没有public类。

4、为什么这个public的类的类名必须和文件名相同

>   答： 是为了方便虚拟机在相应的路径中找到相应的类所对应的字节码文件。

## Main方法

主函数：是一个特殊的函数，作为程序的入口，可以被JVM调用

主函数的定义：
> public：代表着该函数访问权限是最大的

> static：代表主函数随着类的加载就已经存在了

> void：主函数没有具体的返回值

> main：不是关键字，但是一个特殊的单词，能够被JVM识别

> （String[] args）：函数的参数，参数类型是一个数组，该数组中的元素师字符串，字符串数组。main(String[] args) 字符串数组的 此时空数组的长度是0，但也可以在 运行的时候向其中传入参数。

主函数时固定格式的，JVM识别

主函数可以被重载，但是JVM只识别main（String[] args），其他都是作为一般函数。这里面的args知识数组变量可以更改，其他都不能更改。

一个java文件中可以包含很多个类，每个类中有且仅有一个主函数，但是每个java文件中可以包含多个主函数，在运行时，需要指定JVM入口是哪个。例如一个类的主函数可以调用另一个类的主函数。不一定会使用public类的主函数。

## 外部类的访问权限

外部类只能用public和default修饰。

为什么要对外部类或类做修饰呢？

> 1.存在包概念：public 和 default 能区分这个外部类能对不同包作一个划分                       （default修饰的类，其他包中引入不了这个类，public修饰的类才能被import）   
> 
> 2.protected是包内可见并且子类可见，但是当一个外部类想要继承一个protected修饰的非同包类时，压根找不到这个类，更别提几层了
> 
> 3.private修饰的外部类，其他任何外部类都无法导入它。


    //Java中的文件名要和public修饰的类名相同，否则会报错
    //如果没有public修饰的类，则文件可以随意命名
    public class Java中的类文件 {
    
    }
    
    //非公共开类的访问权限默认是包访问权限，不能用private和protected
    //一个外部类的访问权限只有两种，一种是包内可见，一种是包外可见。
    //如果用private修饰，其他类根本无法看到这个类，也就没有意义了。
    //如果用protected，虽然也是包内可见，但是如果有子类想要继承该类但是不同包时，
    //压根找不到这个类，也不可能继承它了，所以干脆用default代替。
    class A{
    
    }
    
## Java包的命名规则

> 以 java.* 开头的是Java的核心包，所有程序都会使用这些包中的类；

> 以 javax.* 开头的是扩展包，x 是 extension 的意思，也就是扩展。虽然 javax.* 是对 java.* 的优化和扩展，但是由于 javax.* 使用的越来越多，很多程序都依赖于 javax.*，所以 javax.* 也是核心的一部分了，也随JDK一起发布。

> 以 org.* 开头的是各个机构或组织发布的包，因为这些组织很有影响力，它们的代码质量很高，所以也将它们开发的部分常用的类随JDK一起发布。

> 在包的命名方面，为了防止重名，有一个惯例：大家都以自己域名的倒写形式作为开头来为自己开发的包命名，例如百度发布的包会以 com.baidu.* 开头，w3c组织发布的包会以 org.w3c.* 开头，微学苑发布的包会以 net.weixueyuan.* 开头……

> 组织机构的域名后缀一般为 org，公司的域名后缀一般为 com，可以认为 org.* 开头的包为非盈利组织机构发布的包，它们一般是开源的，可以免费使用在自己的产品中，不用考虑侵权问题，而以 com.* 开头的包往往由盈利性的公司发布，可能会有版权问题，使用时要注意。


## import的使用

Java import以及Java类的搜索路径
如果你希望使用Java包中的类，就必须先使用import语句导入
语法为：
    
    import package1[.package2…].classname;
    
    package 为包名，classname 为类名。例如：
    
    import java.util.Date; // 导入 java.util 包下的 Date 类
    import java.util.Scanner; // 导入 java.util 包下的 Scanner 类
    import javax.swing.*; // 导入 javax.swing 包下的所有类，* 表示所有类

注意：
> import 只能导入包所包含的类，而不能导入包。
> 为方便起见，我们一般不导入单独的类，而是导入包下所有的类，例如 import java.util.*;。

> Java 编译器默认为所有的 Java 程序导入了 JDK 的 java.lang 包中所有的类（import java.lang.*;），其中定义了一些常用类，如 System、String、Object、Math 等，因此我们可以直接使用这些类而不必显式导入。但是使用其他类必须先导入。

> 前面讲到的”Hello World“程序使用了System.out.println(); 语句，System 类位于 java.lang 包，虽然我们没有显式导入这个包中的类，但是Java 编译器默认已经为我们导入了，否则程序会执行失败。

java类的搜索路径
Java程序运行时要导入相应的类，也就是加载 .class 文件的过程。
假设有如下的 import 语句：

import p1.Test;

> 该语句表明要导入 p1 包中的 Test 类。
> 安装JDK时，我们已经设置了环境变量 CLASSPATH 来指明类库的路径，它的值为 .;%JAVA_HOME%\lib，而 JAVA_HOME 又为 D:\Program Files\jdk1.7.0_71，所以 CLASSPATH 等价于 .;D:\Program Files\jdk1.7.0_71\lib。

> 如果在第一个路径下找到了所需的类文件，则停止搜索，否则继续搜索后面的路径，如果在所有的路径下都未能找到所需的类文件，则编译或运行出错。
>
> 你可以在CLASSPATH变量中增加搜索路径，例如 .;%JAVA_HOME%\lib;C:\javalib，那么你就可以将类文件放在 C:\javalib 目录下，Java运行环境一样会找到。

>用户自己写的类可以通过IDE指定编译后的class文件的输出目录，appclassloader会到指定目录进行类的加载

下面是一个import两种访问权限的类的实例：

    package com.javase.Java中的类.一个包;
    
    public class 全局访问 {
    
    }
    
    
    
    package com.javase.Java中的类.一个包;
    
    
    class 包访问权限 {
    
    }

    package com.javase.Java中的类;
    //import可以导入基础包以及公开的类，需要使用类名的全路径
    //并且在导入某个包.*时，是不会把子包的类给导进来的，这样可以避免导入错误。
    //注意
    //import com.javase.Java中的类.一个包.包访问权限;
    //这个导入会报错，因为这个类没有用public修饰，无法用import导入。
    import com.javase.Java中的类.一个包.全局访问;//可以导入。