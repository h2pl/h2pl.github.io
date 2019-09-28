---
title: Java基础9：解读Java回调机制
date: 2018-04-26 21:23:20
tags:
categories:
	- 后端
	- Java基础
---
本文主要介绍了Java中的回调机制，以及Java多线程中类似回调的机制。

具体代码在我的GitHub中可以找到

https://github.com/h2pl/MyTech

喜欢的话麻烦点下星哈

文章首发于我的个人博客：

https://h2pl.github.io/2018/04/26/javase9

更多关于Java后端学习的内容请到我的CSDN博客上查看：https://blog.csdn.net/a724888

<!-- more -->

## 模块间的调用

本部分摘自https://www.cnblogs.com/xrq730/p/6424471.html


在一个应用系统中，无论使用何种语言开发，必然存在模块之间的调用，调用的方式分为几种：

（1）同步调用

> 同步调用是最基本并且最简单的一种调用方式，类A的方法a()调用类B的方法b()，一直等待b()方法执行完毕，a()方法继续往下走。这种调用方式适用于方法b()执行时间不长的情况，因为b()方法执行时间一长或者直接阻塞的话，a()方法的余下代码是无法执行下去的，这样会造成整个流程的阻塞。

![image](https://images2015.cnblogs.com/blog/801753/201702/801753-20170221201001413-1766758208.png)

（2）异步调用

![image](https://images2015.cnblogs.com/blog/801753/201702/801753-20170221201512429-1532730453.png)

> 异步调用是为了解决同步调用可能出现阻塞，导致整个流程卡住而产生的一种调用方式。类A的方法方法a()通过新起线程的方式调用类B的方法b()，代码接着直接往下执行，这样无论方法b()执行时间多久，都不会阻塞住方法a()的执行。
> 
> 但是这种方式，由于方法a()不等待方法b()的执行完成，在方法a()需要方法b()执行结果的情况下（视具体业务而定，有些业务比如启异步线程发个微信通知、刷新一个缓存这种就没必要），必须通过一定的方式对方法b()的执行结果进行监听。
> 
> 在Java中，可以使用Future+Callable的方式做到这一点，具体做法可以参见我的这篇文章Java多线程21：多线程下其他组件之CyclicBarrier、Callable、Future和FutureTask。

（3）回调

![image](https://images2015.cnblogs.com/blog/801753/201702/801753-20170221205712070-824897248.png)

最后是回调，回调的思想是：  

类A的a()方法调用类B的b()方法
类B的b()方法执行完毕主动调用类A的callback()方法
这样一种调用方式组成了上图，也就是一种双向的调用方式。

## 回调实例：Tom做题

数学老师让Tom做一道题，并且Tom做题期间数学老师不用盯着Tom，而是在玩手机，等Tom把题目做完后再把答案告诉老师。

> 1 数学老师需要Tom的一个引用，然后才能将题目发给Tom。
> 
> 2 数学老师需要提供一个方法以便Tom做完题目以后能够将答案告诉他。
> 
> 3 Tom需要数学老师的一个引用，以便Tom把答案给这位老师，而不是隔壁的体育老师。

回调接口，可以理解为老师接口

        //回调指的是A调用B来做一件事，B做完以后将结果告诉给A，这期间A可以做别的事情。
        //这个接口中有一个方法，意为B做完题目后告诉A时使用的方法。
        //所以我们必须提供这个接口以便让B来回调。
        //回调接口，
        public interface CallBack {
            void tellAnswer(int res);
        }
        
        
数学老师类
        
        //老师类实例化回调接口，即学生写完题目之后通过老师的提供的方法进行回调。
        //那么学生如何调用到老师的方法呢，只要在学生类的方法中传入老师的引用即可。
        //而老师需要指定学生答题，所以也要传入学生的实例。
    public class Teacher implements CallBack{
        private Student student;
    
        Teacher(Student student) {
            this.student = student;
        }
    
        void askProblem (Student student, Teacher teacher) {
            //main方法是主线程运行，为了实现异步回调，这里开启一个线程来操作
            new Thread(new Runnable() {
                @Override
                public void run() {
                    student.resolveProblem(teacher);
                }
            }).start();
            //老师让学生做题以后，等待学生回答的这段时间，可以做别的事，比如玩手机.\
            //而不需要同步等待，这就是回调的好处。
            //当然你可以说开启一个线程让学生做题就行了，但是这样无法让学生通知老师。
            //需要另外的机制去实现通知过程。
            // 当然，多线程中的future和callable也可以实现数据获取的功能。
            for (int i = 1;i < 4;i ++) {
                System.out.println("等学生回答问题的时候老师玩了 " + i + "秒的手机");
            }
        }
    
        @Override
        public void tellAnswer(int res) {
            System.out.println("the answer is " + res);
        }
    }
    
学生接口

        //学生的接口，解决问题的方法中要传入老师的引用，否则无法完成对具体实例的回调。
        //写为接口的好处就是，很多个学生都可以实现这个接口，并且老师在提问题时可以通过
        //传入List<Student>来聚合学生，十分方便。
    public interface Student {
        void resolveProblem (Teacher teacher);
    }

学生Tom


    public class Tom implements Student{
    
        @Override
        public void resolveProblem(Teacher teacher) {
            try {
                //学生思考了3秒后得到了答案，通过老师提供的回调方法告诉老师。
                Thread.sleep(3000);
                System.out.println("work out");
                teacher.tellAnswer(111);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
测试类

    public class Test {
        public static void main(String[] args) {
            //测试
            Student tom = new Tom();
            Teacher lee = new Teacher(tom);
            lee.askProblem(tom, lee);
            //结果
    //        等学生回答问题的时候老师玩了 1秒的手机
    //        等学生回答问题的时候老师玩了 2秒的手机
    //        等学生回答问题的时候老师玩了 3秒的手机
    //        work out
    //        the answer is 111
        }
    }
    
    
## 多线程中的“回调”

Java多线程中可以通过callable和future或futuretask结合来获取线程执行后的返回值。实现方法是通过get方法来调用callable的call方法获取返回值。

其实这种方法本质上不是回调，回调要求的是任务完成以后被调用者主动回调调用者的接口。而这里是调用者主动使用get方法阻塞获取返回值。

    public class 多线程中的回调 {
        //这里简单地使用future和callable实现了线程执行完后
        public static void main(String[] args) throws ExecutionException, InterruptedException {
            ExecutorService executor = Executors.newCachedThreadPool();
            Future<String> future = executor.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    System.out.println("call");
                    TimeUnit.SECONDS.sleep(1);
                    return "str";
                }
            });
            //手动阻塞调用get通过call方法获得返回值。
            System.out.println(future.get());
            //需要手动关闭，不然线程池的线程会继续执行。
            executor.shutdown();

        //使用futuretask同时作为线程执行单元和数据请求单元。
        FutureTask<Integer> futureTask = new FutureTask(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                System.out.println("dasds");
                return new Random().nextInt();
            }
        });
        new Thread(futureTask).start();
        //阻塞获取返回值
        System.out.println(futureTask.get());
    }
    @Test
    public void test () {
        Callable callable = new Callable() {
            @Override
            public Object call() throws Exception {
                return null;
            }
        };
        FutureTask futureTask = new FutureTask(callable);

    }
    }

