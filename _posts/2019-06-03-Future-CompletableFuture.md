---
layout: post
title:  "Future & CompletableFuture"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}

总结Future & CompletableFuture优缺点










## 一、 Future

### 1.1 Future + Callable + ExecutorService 获取执行结果

一般的做法是：使用线程池提交多个Callable任务，返回任务的Future集合，通过遍历来获取每个任务的结果，对结果进行汇总，在很多场景下面使用Future有些乏力，例如以下场景：

* 等待 Future 集合中的所有任务都完成
* 仅等待 Future集合中最快结束的任务完成（有可能因为它们试图通过不同的方式计算同一个值），并返回它的结果
* 任务处理完成时的事件回调

Future + Callable示例：

```java
private static AtomicReference<ExecutorService> serviceRef = new AtomicReference();
    //第一步，初始化线程池
    public static void init(String maxPoolSizeStr) {
        if (serviceRef.get() == null) {
            Class var1 = ThreadPoolExecutorDemo.class;
            synchronized(ThreadPoolExecutorDemo.class) {
                if (serviceRef.get() == null) {
                    int maxPoolSize = 50;
                    if (maxPoolSizeStr != null) {
                        maxPoolSize = Integer.parseInt(maxPoolSizeStr);
                    }

                    //设定固定大小的线程池, 默认线程池大小50
                    serviceRef.set(new ThreadPoolExecutor(maxPoolSize, maxPoolSize, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue()));
                }
            }
        }
    }
    //第二步，使用线程池执行callable任务, 返回future结果集
    public static <T> Map<String, Future<T>> execute(){
        Map<String, Future<T>> resultFutures = new HashMap();
        Map<String, Callable<T>> tasks = createTasks();
        Iterator i$ = tasks.keySet().iterator();

        while(i$.hasNext()) {
            String key = (String)i$.next();
            resultFutures.put(key, serviceRef.get().submit((Callable)tasks.get(key)));
        }

        return resultFutures;
    }
    private static <T> Map<String,Callable<T>> createTasks() {
        //示例创建多个待执行的callable任务,供线程池执行
        Map<String,Callable<T>> taskMap = new HashMap<>();
        taskMap.put("task1", new MyCallable(1));
        taskMap.put("task2",new MyCallable(2));
        return taskMap;
    }
    //第三步, 提供使用完成之后停止线程池的方法
    public static void shutdown() {
        if (serviceRef.get() != null) {
            synchronized(ThreadPoolExecutorDemo.class) {
                if (serviceRef.get() != null) {
                    serviceRef.get().shutdown();
                    serviceRef.set(null);
                }
            }
        }
    }
```

### 1.2 FutureTask + Callable + ExecutorService/Thread 获取执行结果

FutureTask定义：

```java
public class FutureTask<V> implements RunnableFuture<V> {}

public interface RunnableFuture<V> extends Runnable, Future<V> {}
```

FutureTask实现了RunnableFuture接口，RunnableFuture接口又继承于Runnable和Future接口，也就是说FutureTask同时具有Runnable和Future的优点

* 普通Future可以获取返回值，但是必须通过线程池提交Callable来获取Future
* 普通Runnable不依赖线程池，但是无法获取返回值
* FutureTask则可以直接通过Thread来执行，也可以通过线程池来执行，不用把任务提交给线程池，且可以异步获取执行结果

FutureTask + Callable示例：

```java
import java.util.concurrent.*;

public class FutureTaskDemo {
    public static void main(String[] args) {
        //第一种方式，通过线程池提交FutureTask包装后的Callable，FutureTask自动接收结果
        ExecutorService executor = Executors.newCachedThreadPool();
        MyCallable myCallable = new MyCallable();
        FutureTask<Integer> futureTask = new FutureTask<>(myCallable);
        executor.submit(futureTask);
        executor.shutdown();

        //第二种方式，直接通过Thread执行FutureTask包装后的Callable
        // 注意这种方式和第一种方式效果是类似的，只不过一个使用的是ExecutorService，一个使用的是Thread
        /*Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        Thread thread = new Thread(futureTask);
        thread.start();*/

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }

        System.out.println("主线程在执行任务");

        try {
            System.out.println("task运行结果"+futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        System.out.println("所有任务执行完毕");
    }
}

class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++) {
            sum += i;
        }
        return sum;
    }
}
```

### 1.3 CompletionService获取先执行完成的任务结果

> CompletionService与ExecutorService类似都可以用来执行线程池的任务，CompletionService的一个实现是ExecutorCompletionService，它在多任务处理时会把处理结果按照完成的先后顺序加入到BlockingQueue，通过take方法可以获取到最先执行完成的结果

```java
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.CompletionService;
import java.util.concurrent.ExecutorCompletionService;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class CompletionServiceDemo{
    public static void main(String[] args)  {
        Long start = System.currentTimeMillis();
        //定义线程池
        ExecutorService exs = Executors.newFixedThreadPool(5);
        try {
            int taskCount = 10;
            //结果集
            List<Integer> list = new ArrayList<>();
            //1.定义CompletionService
            CompletionService<Integer> completionService = new ExecutorCompletionService<>(exs);
            List<Future<Integer>> futureList = new ArrayList<>();
            //2.添加任务
            for(int i=0;i<taskCount;i++){
                futureList.add(completionService.submit(new MyCallable(i+1)));
            }
            //==================结果归集===================
            //方法1：future是提交时返回的，遍历queue则按照任务提交顺序，获取结果
            /*for (Future<Integer> future : futureList) {
                System.out.println("====================");
                Integer result = future.get();//线程在这里阻塞等待该任务执行完毕,按照
                System.out.println("任务result="+result+"获取到结果!"+new Date());
                list.add(result);
            }*/

            //方法2.使用内部阻塞队列的take()
            for(int i=0;i<taskCount;i++){
                //采用completionService.take()，内部维护阻塞队列，任务先完成的先获取到
                Integer result = completionService.take().get();
                System.out.println("任务i=="+result+"完成!"+new Date());
                list.add(result);
            }
            System.out.println("list="+list);
            System.out.println("总耗时="+(System.currentTimeMillis()-start));
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            exs.shutdown();//关闭线程池
        }
    }

    static class MyCallable implements Callable<Integer>{
        Integer i;

        public MyCallable(Integer i) {
            super();
            this.i=i;
        }

        @Override
        public Integer call() throws Exception {
            Thread.sleep(1000);
            System.out.println("线程："+Thread.currentThread().getName()+"任务i="+i+",执行完成！");
            return i;
        }
    }
}
```

## 二、CompletableFuture

### 2.1 

## 三、Future|FutureTask|CompletionService|CompletableFuture对比

|                    | Futrue                                  | FutureTask                                                                  | CompletionService                                            | CompletableFuture                               |
| ------------------ | --------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------- |
| 原理             | Futrue接口                            | 接口RunnableFuture的唯一实现类，RunnableFuture接口继承自Future<V>+Runnable: | 内部通过阻塞队列+FutureTask接口                    | JDK8实现了Future<T>, CompletionStage<T>2个接口 |
| 多任务并发执行 | 支持                                  | 支持                                                                      | 支持                                                       | 支持                                          |
| 获取任务结果的顺序 | 支持任务完成先后顺序          |  未知                                                                    |  支持任务完成的先后顺序                          | 支持任务完成的先后顺序               |
| 异常捕捉       |  自己捕捉                          |  自己捕捉                                                              |  自己捕捉                                               | 源生API支持，返回每个任务的异常   |
| 建议             | CPU高速轮询，耗资源，可以使用，但不推荐 |  功能不对口，并发任务这一块多套一层，不推荐使用。  |  推荐使用，没有JDK8CompletableFuture之前最好的方案，没有质疑 | API极端丰富，配合流式编程，速度飞起，推荐使用！ |

## 参考

[*****多线程并发执行任务，取结果归集。终极总结：Future、FutureTask、CompletionService、CompletableFuture](https://www.cnblogs.com/dennyzhangdd/p/7010972.html)
