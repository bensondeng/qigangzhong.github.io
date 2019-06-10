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

> CompletableFuture类实现了CompletionStage接口，以及Future接口，如果不指定自定义的线程池，则默认使用ForkJoinPool.commonPool()当做线程池

### 2.1 获取任务执行结果的方式

获取任务执行结果的方式有以下几种：

```java
//阻塞直到获取到结果或者抛出异常，会向外部抛出ExecutionException
public T get();
//阻塞直到获取到结果或者抛出异常或超时，会向外部抛出ExecutionException、TimeoutException
public T get(long timeout, TimeUnit unit);
//如果计算完成没有问题返回计算后的值
//如果任务执行异常则获取到默认的值，该方法不会再向外抛出异常
public T getNow(T valueIfAbsent);
//阻塞直到获取到结果或者抛出异常，会向外抛出CompletionException
public T join();
```

### 2.2 示例

```java
import java.util.concurrent.*;
public class CompletableFutureDemo {
    public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {
        //completedFutureExample();
        //runAsyncExample();
        //thenApplyExample();
        //thenApplyWithExecutorExample();
        //thenAcceptExample();
        //completeExceptionallyExample();
        //cancelExample();
        //thenComposeExample();
        //doSthWhenTasksCompleted();
    }

    //region 1、 创建一个完成的CompletableFuture
    private static void completedFutureExample() {
        //其实就是使用常量组装一个CompletableFuture而已
        CompletableFuture<String> cf = CompletableFuture.completedFuture("message");
        Object result = cf.getNow(null);
        System.out.println(result);
    }
    //endregion

    //region 2、运行一个简单的异步阶段
    private static void runAsyncExample() throws InterruptedException {
        CompletableFuture<Void> cf = CompletableFuture.runAsync(()->{
            System.out.println("一步线程是否是daemon线程:"+Thread.currentThread().isDaemon());
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("done...");
        });

        System.out.println("结束了吗？"+cf.isDone());

        TimeUnit.SECONDS.sleep(3);

        System.out.println("结束了吗？"+cf.isDone());
    }
    //endregion

    //region 3、在前一个阶段上应用函数
    private static void thenApplyExample() {
        /*CompletableFuture cf = CompletableFuture.completedFuture("message").thenApply(v-> {
            return v.toUpperCase();
        });*/
        //CompletableFuture cf = CompletableFuture.completedFuture("message").thenApply(v->v.toUpperCase());
        CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(String::toUpperCase);
        //thenApplyAsync异步方法直接通过get无法获取到结果，thenApply同步方法可以直接获取到结果
        System.out.println(cf.getNow(null));
        //通过join来阻塞获取执行结果
        System.out.println(cf.join());
    }
    //endregion

    //region 5、使用定制的Executor在前一个阶段上异步应用函数
    static ExecutorService executorService = Executors.newFixedThreadPool(3, new ThreadFactory() {
        int count = 1;
        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r,"my-executor-"+count);
        }
    });

    private static void thenApplyWithExecutorExample() {
        CompletableFuture cf = CompletableFuture.completedFuture("message").thenApplyAsync(v->{
            System.out.println("当前异步线程名称："+Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return v.toUpperCase();
        }, executorService);

        System.out.println("直接通过get拿到的结果："+cf.getNow(null));

        System.out.println("通过join拿到的结果："+cf.join());
    }
    //endregion

    //region 6、消费前一阶段的结果
    private static void thenAcceptExample() {
        StringBuilder sb = new StringBuilder();
        //CompletableFuture.completedFuture("thenAccept message").thenAccept(v->sb.append(v));
        CompletableFuture<Void> cf = CompletableFuture.completedFuture("thenAccept message").thenAcceptAsync(v->sb.append(v));
        //带async的操作必须通过join阻塞等待执行结束
        cf.join();
        System.out.println(sb.toString());
    }
    //endregion

    //region 7、捕获异常
    private static void completeExceptionallyExample() {
        CompletableFuture<String> cf = CompletableFuture.completedFuture("message").thenApplyAsync(v->{
            if(true) {
                throw new RuntimeException("exception test!");
            }
            return v.toUpperCase();
        }).exceptionally(e->{
            System.out.println(e.getMessage());
            return "出错了";
        });
        System.out.println(cf.join());
    }
    //endregion

    //region 8、取消任务
    private static void cancelExample() {
        CompletableFuture cf1 = CompletableFuture.completedFuture("message").thenApplyAsync(v->{
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return v.toUpperCase();
        });
        CompletableFuture cf2 = cf1.exceptionally(e->{
            return "出错了";
        });
        boolean isCanceled = cf1.cancel(true);
        System.out.println("cf1被取消了吗？"+isCanceled);
        System.out.println("cf1完成了吗？"+cf1.isDone());

        System.out.println("cf2执行结果："+cf2.join());
    }
    //endregion

    //region 9、将前一个任务的执行结果(包括异常)作为参数传递给后一个任务
    private static void thenComposeExample() {
        long start = System.currentTimeMillis();

        /*CompletableFuture cf = CompletableFuture.supplyAsync(() -> 3).thenCompose(i-> CompletableFuture.supplyAsync(()->i+1));
        System.out.println(cf.join());*/

        CompletableFuture cf = CompletableFuture.supplyAsync(() -> {
            /*if(true){
                throw new RuntimeException("出现异常了");
            }*/

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return 3;
        }).handleAsync((v,e)->{
            if(e==null){
                System.out.println("前一个任务没有出现异常，处理结果为："+v);
            }else {
                System.out.println("前一个任务执行出现异常：" + e.getMessage());
            }

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }

            return v;
        });
        System.out.println(cf.join());

        long end = System.currentTimeMillis();
        System.out.println("执行结束，耗时：" + (end - start));
    }
    //endregion

    //region 10、任务并行处理完成之后做一些事情(结果合并，并行任务完成后回调等)
    private static void doSthWhenTasksCompleted() {
        long start = System.currentTimeMillis();
        String original = "Message";
        CompletableFuture cf1 = CompletableFuture.completedFuture(original).thenApplyAsync(v->{
            //模拟执行时间2s
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return v.toUpperCase();
        });
        CompletableFuture cf2 = CompletableFuture.completedFuture(original).thenApplyAsync(v->{
            //模拟执行时间1s
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return v.toLowerCase();
        });

        //拿到最先执行完成的任务结果
        /*CompletableFuture cf3 = cf1.applyToEither(cf2,v->{
            return v+" from applyToEither";
        });
        System.out.println(cf3.join());*/

        //执行结束仅仅做一些事情，但不处理并行任务的返回结果
        /*CompletableFuture cf3 = cf1.runAfterBoth(cf2,()->{
            System.out.println("执行结束，但是runAfterBoth拿不到执行结果");
        });
        System.out.println(cf3.join());*/

        //执行结束获取每个任务的返回结果，进行处理
        /*CompletableFuture cf3 = cf1.thenAcceptBoth(cf2,(v1,v2)->{
            System.out.println(String.format("两个任务都执行结束了，结果v1=%s，v2=%s",v1,v2));
        });
        System.out.println(cf3.join());*/

        //执行结束合并两个任务的执行结果
        /*CompletableFuture cf3 = cf1.thenCombineAsync(cf2,(v1,v2)->{
            return "合并执行结果："+v1+v2;
        });
        System.out.println(cf3.join());*/

        //拿到最先执行完成的任务结果（类似applyToEither）
        /*CompletableFuture cf3 = CompletableFuture.anyOf(cf1,cf2).whenComplete((v,e)->{
            if(e==null){
                System.out.println("取到其中一个执行结果："+v);
            }
        });
        System.out.println(cf3.join());*/

        CompletableFuture cf3 = CompletableFuture.allOf(cf1,cf2).whenComplete((v,e)->{
            System.out.println(String.format("两个任务全部执行完成，cf1结果：%s，cf2结果：%s",cf1.getNow(null),cf2.getNow(null)));
        });
        System.out.println(cf3.join());

        long end = System.currentTimeMillis();
        System.out.println("执行时间ms："+(end-start));
    }
    //endregion

}
```

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

[20 个使用 Java CompletableFuture的例子](http://www.importnew.com/28319.html)

[completablefuture-examples](https://github.com/manouti/completablefuture-examples)

[CompletableFuture与CompletionStage使用](https://my.oschina.net/JackieRiver/blog/2054472)
