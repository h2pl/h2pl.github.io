---
title: Java基础6：代码块与代码加载顺序
date: 2018-04-24 11:31:00
tags:
categories:
	- 后端
	- Java基础
---
本文主要介绍了三种代码块的特性和使用方法。

具体代码在我的GitHub中可以找到

https://github.com/h2pl/MyTech

喜欢的话麻烦点下星哈

文章首发于我的个人博客：

https://h2pl.github.io/2018/04/24/javase6

更多关于Java后端学习的内容请到我的CSDN博客上查看：

https://blog.csdn.net/a724888
<!-- more -->

代码块：用{}包围的代码

java中的代码块按其位置划分为四种：

## 局部代码块

> 位置：局部位置（方法内部）

> 作用：限定变量的生命周期，尽早释放，节约内存

> 调用：调用其所在的方法时执行

     

     public class 局部代码块 {
    @Test
    public void test (){
        B b = new B();
        b.go();
    }
    }
    class B {
        B(){}
        public void go() {
            //方法中的局部代码块，一般进行一次性地调用，调用完立刻释放空间，避免在接下来的调用过程中占用栈空间
            //因为栈空间内存是有限的，方法调用可能会会生成很多局部变量导致栈内存不足。
            //使用局部代码块可以避免这样的情况发生。
            {
                int i = 1;
                ArrayList<Integer> list = new ArrayList<>();
                while (i < 1000) {
                    list.add(i ++);
                }
                for (Integer j : list) {
                    System.out.println(j);
                }
                System.out.println("gogogo");
            }
            System.out.println("hello");
        }
    }

## 构造代码块

 >位置：类成员的位置，就是类中方法之外的位置
 
 >作用：把多个构造方法共同的部分提取出来，共用构造代码块
 
 >调用：每次调用构造方法时，都会优先于构造方法执行，也就是每次new一个对象时自动调用，对  对象的初始化

    class A{
        int i = 1;
        int initValue;//成员变量的初始化交给代码块来完成
        {
            //代码块的作用体现于此：在调用构造方法之前，用某段代码对成员变量进行初始化。
            //而不是在构造方法调用时再进行。一般用于将构造方法的相同部分提取出来。
            //
            for (int i = 0;i < 100;i ++) {
                initValue += i;
            }
        }
        {
            System.out.println(initValue);
            System.out.println(i);//此时会打印1
            int i = 2;//代码块里的变量和成员变量不冲突，但会优先使用代码块的变量
            System.out.println(i);//此时打印2
            //System.out.println(j);//提示非法向后引用，因为此时j的的初始化还没开始。
            //
        }
        {
            System.out.println("代码块运行");
        }
        int j = 2;
        {
            System.out.println(j);
            System.out.println(i);//代码块中的变量运行后自动释放，不会影响代码块之外的代码
        }
        A(){
            System.out.println("构造方法运行");
        }
    }
    public class 构造代码块 {
        @Test
        public void test() {
            A a = new A();
        }
    }

## 静态代码块

     位置：类成员位置，用static修饰的代码块

     作用：对类进行一些初始化  只加载一次，当new多个对象时，只有第一次会调用静态代码块，因为，静态代码块                  是属于类的，所有对象共享一份

     调用: new 一个对象时自动调用

     public class 静态代码块 {

    @Test
    public void test() {
        C c1 = new C();
        C c2 = new C();
        //结果,静态代码块只会调用一次，类的所有对象共享该代码块
        //一般用于类的全局信息初始化
        //静态代码块调用
        //代码块调用
        //构造方法调用
        //代码块调用
        //构造方法调用
    }

    }
    class C{
        C(){
            System.out.println("构造方法调用");
        }
        {
            System.out.println("代码块调用");
        }
        static {
            System.out.println("静态代码块调用");
        }
    }

执行顺序       静态代码块 ----->   构造代码块 ------->   构造方法

    