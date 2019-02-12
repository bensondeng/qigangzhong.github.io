---
layout: post
title:  "线程基础知识总结"
categories: thread
tags:  thread
author: 刚子
---

* content
{:toc}


## 前言


总结java线程基础知识

##  课程目录
1. 创建线程的三种方式
2. Thread方法介绍










## 创建线程的三种方式

### 1. 通过实现 `Runnable` 接口

```java
class RunnableDemo implements Runnable {
   private Thread t;
   private String threadName;
   
   RunnableDemo( String name) {
      threadName = name;
      System.out.println("Creating " +  threadName );
   }
   
   public void run() {
      System.out.println("Running " +  threadName );
      try {
         for(int i = 4; i > 0; i--) {
            System.out.println("Thread: " + threadName + ", " + i);
            // 让线程睡眠一会
            Thread.sleep(50);
         }
      }catch (InterruptedException e) {
         System.out.println("Thread " +  threadName + " interrupted.");
      }
      System.out.println("Thread " +  threadName + " exiting.");
   }
   
   public void start () {
      System.out.println("Starting " +  threadName );
      if (t == null) {
         t = new Thread (this, threadName);
         t.start ();
      }
   }
}
 
public class TestThread {
 
   public static void main(String args[]) {
      RunnableDemo R1 = new RunnableDemo( "Thread-1");
      R1.start();
      
      RunnableDemo R2 = new RunnableDemo( "Thread-2");
      R2.start();
   }   
}
```

### 2. 通过继承 `Thread` 类本身

```java
class ThreadDemo extends Thread {
   private Thread t;
   private String threadName;
   
   ThreadDemo( String name) {
      threadName = name;
      System.out.println("Creating " +  threadName );
   }
   
   public void run() {
      System.out.println("Running " +  threadName );
      try {
         for(int i = 4; i > 0; i--) {
            System.out.println("Thread: " + threadName + ", " + i);
            // 让线程睡眠一会
            Thread.sleep(50);
         }
      }catch (InterruptedException e) {
         System.out.println("Thread " +  threadName + " interrupted.");
      }
      System.out.println("Thread " +  threadName + " exiting.");
   }
   
   public void start () {
      System.out.println("Starting " +  threadName );
      if (t == null) {
         t = new Thread (this, threadName);
         t.start ();
      }
   }
}
 
public class TestThread {
 
   public static void main(String args[]) {
      ThreadDemo T1 = new ThreadDemo( "Thread-1");
      T1.start();
      
      ThreadDemo T2 = new ThreadDemo( "Thread-2");
      T2.start();
   }   
}
```

### 3. 通过 `Callable` 配合 `Future` 或者 `FutureTask` 创建线程

> 通过实现Runnable接口或者继承Thread类这两种方式来启动线程都有个问题，就是无法获取执行结果，实现Callable接口可以解决这个问题
> Callable一般配合ExecutorService来使用，通过泛型参数来定义返回的数据类型，ExecutorService提供的submit方法可以执行任务
> >  <T> Future<T> submit(Callable<T> task);
> 返回的Future<T>对象提供了一些有用的方法，其中包含获取执行结果的get方法
> > cancel方法用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
> > isCancelled方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true
> > isDone方法表示任务是否已经完成，若任务完成，则返回true
> > get()方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回
> > get(long timeout, TimeUnit unit)用来获取执行结果，如果在指定时间内，还没获取到结果，就抛出异常

```java
import java.util.concurrent.*;
public class Test {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Task task = new Task();
        Future<Integer> future = executorService.submit(task);
        executorService.shutdown();
        
        System.out.println("主线程在执行任务...");
        try {
            Thread.sleep(2000);
        } catch(InterruptedException ex) {
            ex.printStackTrace();
        }
         
        try {
            System.out.println("task运行结果:"+future.get());
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        } catch (ExecutionException ex) {
            ex.printStackTrace();
        }  
        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在执行任务...");
        //模拟任务耗时
        Thread.sleep(5000);
        return 1000;
    }
}
```

```java
import java.util.concurrent.*;
public class Test {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        executorService.submit(futureTask);
        executorService.shutdown();
        
        System.out.println("主线程在执行任务...");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
         
        try {
            System.out.println("task运行结果:"+futureTask.get());
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        } catch (ExecutionException ex) {
            ex.printStackTrace();
        }
         
        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在执行任务...");
        //模拟任务耗时
        Thread.sleep(5000);
        return 1000;
    }
}
```



## 参考

[Java 多线程编程](http://www.runoob.com/java/java-multithreading.html)

[在Java中使用Callable、Future进行并行编程](https://segmentfault.com/a/1190000012291442)

[原始博客作者站点](https://gaohaoyang.github.io/)



