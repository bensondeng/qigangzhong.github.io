---
layout: post
title:  "Spring WebFlux 教程（译）"
categories: java-basics
tags:  java-basics
author: Lokesh Gupta
---

* content
{:toc}

> 在Spring5.0中加入了反应堆栈(reactive-stack)web框架Spring WebFlux。它是完全非阻塞的，支持响应流([reactive streams](http://www.reactive-streams.org/))背压(back pressure)，并且可以运行在诸如Netty，Undertow和Servlet3.1+的容器中。在这篇spring webflux教程中，我们将学习响应式编程背后的基础概念、webflux apis和一个完整的hello world示例。








## 1. 响应式编程

响应式编程是一种编程范式，它增强了异步、非阻塞、事件驱动的数据处理方式。响应式编程包括将数据和事件建模为可被观测的数据流，以及实现数据处理程序来响应数据流中的变化。

在深入了解响应式编程之前，首先来了解一下阻塞和非阻塞请求处理的区别。

### 1.1 阻塞 vs 非阻塞（异步）请求处理

#### 1.1.1 阻塞请求处理

在传统的MVC应用中，当一个请求来到服务端之后，一个servlet线程就会被创建。它把请求委托给诸如数据库访问这样的I/O处理工作线程。当工作线程工作繁忙的时候，servlet线程（请求线程）保持等待状态所以造成了阻塞。这种方式也叫同步请求处理。

![blocking-request-processing.png](/images/spring/blocking-request-processing.png)

由于服务器可能只有有限数量的线程，这就限制了服务器处理最大负载请求数量的能力。

#### 1.1.2 非阻塞请求处理

在非阻塞或异步请求处理中，没有线程处于等待状态。通常只有一个请求线程来接收请求。

所有请求进来的时候都带有时间处理器(event handler)和回调(callback)信息。请求线程将进入的请求委托给一个线程池（通常有少量的线程），线程池委托请求给处理函数之后立即开始处理其它来自请求线程的请求。

当处理函数完成之后，线程池中的一个线程收集响应信息传递给回调(callback)函数。

![non-blocking-request-processing.png](/images/spring/non-blocking-request-processing.png)

非阻塞的线程提高了应用的性能表现，少量的线程意味着更少的内存开销以及更少的上下文切换。

### 1.2 什么是响应式编程？

“响应”这个术语指的是围绕响应事件而构建的编程模型。它是围绕“发布-订阅”模式（观察者模式）而构建的。在响应式的编程风格中，我们请求一个资源之后就处理其它事情了，当数据好了以后，我们获取到通知和数据来告知回调函数，在回调函数里面，我们根据应用/用户需要来处理响应。

需要记住的一个重要的事情就是背压（back pressure）。在非阻塞代码中，控制事件的速度以防生产者快速压垮下游变得很重要。

响应式web编程对于那些具有流数据的应用，消费流数据并流转到用户方的客户端都是极好的。但对于开发传统的CRUD的应用不是很好。如果你即将开发的是一个具有大量数据，类似于下一代Facebook或Twitter的应用，那响应式API也许正是你需要的。

## 2. Reactive Streams API

新的Reactive Streams API由来自Netflix，Pivotal，Lightbend，RedHat，Twitter，Oracle等的工程师创造的，现在已经是Java9的一部分。它定义了4类接口：

* [Publisher](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.2/api/src/main/java/org/reactivestreams/Publisher.java)：根据订阅方（subscriber）的需求发射一系列的事件给订阅方。一个publisher可以服务于多个订阅方。

它只有一个方法：

```java
public interface Publisher<T>
{
    public void subscribe(Subscriber<? super T> s);
}
```

* [Subscriber](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.2/api/src/main/java/org/reactivestreams/Subscriber.java)：接收和处理由publisher发射的事件。注意除非调用`Subscription#request(long)`来发出需求信号，否则不会收到通知。

它有4个方法来处理各种接收到的响应。

```java
public interface Subscriber<T>
{
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
```

* [Subscription](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.2/api/src/main/java/org/reactivestreams/Subscription.java)：定义了一个`Publisher`和一个`Subscriber`之间的一对一关系。它只能被一个`Subscriber`使用一次。被用来发出数据请求和取消的信号（允许资源清理）。

```java
public interface Subscription<T>
{
    public void request(long n);
    public void cancel();
}
```

* [Processor](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.2/api/src/main/java/org/reactivestreams/Processor.java)：代表一个由`Publisher`和`Subscriber`组成的处理阶段，遵守两者的接口约定。

```java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R>
{
}
```

> 两个比较流行的响应式流(reactive streams)的实现是[RxJava](https://github.com/ReactiveX/RxJava)、[Project Reactor](https://projectreactor.io/)

## 3. 什么是Spring WebFlux？

Spring WebFlux是SpringMVC的并行版本，且支持完全非阻塞的响应式流。支持背压概念，使用Netty作为内置服务器来运行响应式应用。如果你对Spring MVC编程风格很熟悉，你可以很容易上手webflux。

Spring WebFlux使用project reactor作为响应式库。reactor是一个响应式流的库，所有的操作都支持非阻塞背压。它是在与Spring的密切合作之下开发的。

Spring WebFlux很大程度上使用了两个publisher：

* Mono：返回0或1个元素。

```java
Mono<String> mono = Mono.just("Alex");
Mono<String> mono = Mono.empty();
```

* Flux：返回0...N个元素。Flux可以连续无止境，它可以一直发射元素。而且它可以返回一系列的元素并在返回所有元素的时候发送一个完成通知。

```java
Flux<String> flux = Flux.just("A", "B", "C");
Flux<String> flux = Flux.fromArray(new String[]{"A", "B", "C"});
Flux<String> flux = Flux.fromIterable(Arrays.asList("A", "B", "C"));

//To subscribe call method

flux.subscribe();
```

## 参考

[Spring WebFlux Tutorial](https://howtodoinjava.com/spring-webflux/spring-webflux-tutorial/)

[Web on Reactive Stack](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)

[如何形象的描述反应式编程中的背压(Backpressure)机制？](https://www.zhihu.com/question/49618581/answer/237078934)

[关于RxJava最友好的文章——背压（Backpressure）](https://www.jianshu.com/p/2c4799fa91a4)
