---
layout: post
title:  "ThreadLocal、volatile"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}


## 前言

总结java线程基础知识

##  课程目录
* ThreadLocal、volatile









## 一. ThreadLocal

### 1. 介绍

> 线程本地变量，存储变量在每个线程中独立的副本
> 
> 适用场景: 
>
> > 类似于session这种贯穿线程生命周期使用的变量

### 2. 方法

```java
public T get() { }
public void set(T value) { }
public void remove() { }
protected T initialValue() { } //ThreadLocal没有被当前线程赋值时或当前线程刚调用remove方法后调用get方法，返回此方法值
```

> 调用get方法之前必须先调用get，否则会报空指针异常  
> 或者定义变量时设置初始值  

```java
private static ThreadLocal<String> threadLocal = new ThreadLocal<String>(){
    @Override
    protected String initialValue() {
        return "hello"; //给一个初始值，直接调用get方法时返回初始值而不是默认的null
    }
};

//或者
private static ThreadLocal<String> tls = ThreadLocal.withInitial(() -> "hello");
```

### 3. 内存泄漏问题解决

> 关于ThreadLocalMap<ThreadLocal, Object>弱引用问题：
> 
> 当线程没有结束，但是ThreadLocal已经被回收，则可能导致线程中存在ThreadLocalMap<null, Object>的键值对，造成内存泄露。（ThreadLocal被回收，ThreadLocal关联的线程共享变量还存在）。
> 
> 虽然ThreadLocal的get，set方法可以清除ThreadLocalMap中key为null的value，但是get，set方法在内存泄露后并不会必然调用，所以为了防止此类情况的出现，我们有两种手段。
> 
> 1、使用完线程共享变量后，显示调用`ThreadLocalMap.remove`方法清除线程共享变量；
> 
> 2、JDK建议ThreadLocal定义为`private static`，这样ThreadLocal的弱引用问题则不存在了。

## 二、volatile

### 1. 介绍

> 内存模型的三个概念:  
> **原子性**  volatile不能保证原子性  
> **可见性**  volatile保证不同线程的可见性  
> **有序性**  volatile可以一定程度保证有序性，可以防止指令重排序
>
> 下面这段话摘自《深入理解Java虚拟机》：  
>　　“观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令”，lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障会提供3个功能：
> * 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
> * 它会强制将对缓存的修改操作立即写入主存；
> * 如果是写操作，它会导致其他CPU中对应的缓存行无效。
>
> 在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。因此，经过分析我们可以得出如下结论：
> * Lock前缀的指令会引起处理器缓存写回内存；
> * 一个处理器的缓存回写到内存会导致其他处理器的缓存失效；
> * 当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。


### 2. 应用场景

* 状态标记

```java
volatile boolean flag = false;
 
while(!flag){
    doSomething();
}
 
public void setFlag() {
    flag = true;
}
```

* double check

```java
class Singleton{
    private volatile static Singleton instance = null;
     
    private Singleton() {}
     
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

## 参考
[ThreadLocal用法详解和原理](https://www.cnblogs.com/coshaho/p/5127135.html)

[Java并发编程：深入剖析ThreadLocal](https://www.cnblogs.com/dolphin0520/p/3920407.html)

[Java并发编程：volatile关键字解析](https://www.cnblogs.com/dolphin0520/p/3920373.html)

[让你彻底理解volatile](https://www.jianshu.com/p/157279e6efdb)

[*****正确使用 Volatile 变量](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)