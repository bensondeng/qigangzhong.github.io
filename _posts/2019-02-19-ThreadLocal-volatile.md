---
layout: post
title:  "wait、notify、notifyAll"
categories: thread
tags:  thread
author: 刚子
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




## 参考
[ThreadLocal用法详解和原理](https://www.cnblogs.com/coshaho/p/5127135.html)

[Java并发编程：深入剖析ThreadLocal](https://www.cnblogs.com/dolphin0520/p/3920407.html)
