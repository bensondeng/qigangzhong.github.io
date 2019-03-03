---
layout: post
title:  "线程池"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}


## 前言

总结java线程基础知识

##  课程目录
* ThreadPoolExecutor







### ThreadPoolExecutor

#### 构造函数
```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue);
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
```
1. 几个核心参数概念:
* corePoolSize  
池大小，如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务，当线程池中任务数量到达改值时，新的任务会放到队列中
* maximumPoolSize  
最大线程数，超过这个数量就拒绝服务
* keepAliveTime  
表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0
* unit  
参数keepAliveTime的时间单位，有7种取值
* workQueue  
一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：
  * ArrayBlockingQueue(必须指定队列大小)、PriorityBlockingQueue  使用较少
  * LinkedBlockingQueue: 基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE
  * SynchronousQueue: 这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务
* threadFactory  
线程工厂，主要用来创建线程
* handler  
表示当拒绝处理任务时的策略，有以下四种取值:
  * ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
  * ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
  * ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
  * ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务

2. 注意事项

> * ThreadPoolExecutor将根据 corePoolSize和 maximumPoolSize设置的边界自动调整池大小
当新任务在方法 execute(java.lang.Runnable) 中提交时，如果运行的线程少于 corePoolSize，则创建新线程来处理请求，即使其他辅助线程是空闲的。
> * 如果运行的线程多于 corePoolSize 而少于 maximumPoolSize，则仅当队列满时才创建新线程
如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，超过数量的任务将被拒绝。
> * 如果设置的 corePoolSize 和 maximumPoolSize 相同，则创建了固定大小的线程池。
> * 如果将 maximumPoolSize 设置为基本的无界值（如 Integer.MAX_VALUE），则允许池适应任意数量的并发任务。
> * 在大多数情况下，核心和最大池大小仅基于构造函数来设置，不过也可以使用 setCorePoolSize(int) 和 setMaximumPoolSize(int) 进行动态更改。


> 并不是先加入任务就一定会先执行，假设队列大小为 4，corePoolSize为2，maximumPoolSize为6，那么当加入15个任务时，
执行的顺序类似这样：首先执行任务 1、2，然后任务3~6被放入队列。这时候队列满了，任务7、8、9、10 会被马上执行，而任务 11~15 则会抛出异常。最终顺序是：1、2、7、8、9、10、3、4、5、6。  
> 当然这个过程是针对指定大小的ArrayBlockingQueue<Runnable>来说，如果是默认的LinkedBlockingQueue<Runnable>，因为该队列无大小限制，maximumPoolSize会无效，队列大小不受控制，会有资源耗尽的风险。

> 最多能执行多少个任务？？？ maximumPoolSize+队列长度

简单验证
```java
//简单示例
public static void main(String[] args) {
    ThreadPoolExecutor executor=new ThreadPoolExecutor(
            2,
            6,
            60,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(4));

    for(int i=0;i<15;i++){
        MyRunnable myRunnable = new MyRunnable(i);
        executor.execute(myRunnable);
        System.out.println("线程池中线程数目："+executor.getPoolSize()+
                "，队列中等待执行的任务数目："+executor.getQueue().size()+
                "，已执行完别的任务数目："+executor.getCompletedTaskCount());
    }

    executor.shutdown();
}

class MyRunnable implements Runnable {
    private int taskNum;

    public MyRunnable(int num) {
        this.taskNum = num;
    }

    @Override
    public void run() {
        System.out.println("正在执行task "+taskNum);
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task "+taskNum+"执行完毕");
    }
}
```

#### 线程状态定义
```java
private static final int RUNNING    = -1 << COUNT_BITS;
//如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
//当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态
private static final int TERMINATED =  3 << COUNT_BITS;
```

### Executors

在java doc中，并不提倡我们直接使用ThreadPoolExecutor来创建线程池，而是使用Executors类中提供的几个静态方法来创建线程池(底层其实还是调用ThreadPoolExecutor来创建线程池)
```java
//1. 创建一个可缓存线程池，应用中存在的线程数可以无限大
ExecutorService newCachedThreadPool = Executors.newCachedThreadPool();
System.out.println("****************************newCachedThreadPool*******************************");
for(int i=0;i<4;i++)
{
    final int index=i;
    newCachedThreadPool.execute(new FourThreadPoolsRunnableImpl(index));
}


//2. 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待，corePoolSize和maximumPoolSize值相等
ExecutorService newFixedThreadPool = Executors.newFixedThreadPool(2);
System.out.println("****************************newFixedThreadPool*******************************");
for(int i=0;i<4;i++)
{
    final int index=i;
    newFixedThreadPool.execute(new FourThreadPoolsRunnableImpl(index));
}


//3. 创建一个定长线程池，支持定时及周期性任务执行
ScheduledExecutorService newScheduledThreadPool = Executors.newScheduledThreadPool(2);
System.out.println("****************************newFixedThreadPool*******************************");
for(int i=0;i<4;i++)
{
    final int index=i;
    //延迟三秒执行
    newScheduledThreadPool.schedule(new FourThreadPoolsRunnableImpl(index),3, TimeUnit.SECONDS);
}


//4. 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行，corePoolSize和maximumPoolSize都被设置为1
ExecutorService newSingleThreadExecutor = Executors.newSingleThreadExecutor();
System.out.println("****************************newFixedThreadPool*******************************");
for(int i=0;i<4;i++)
{
    final int index=i;
    newSingleThreadExecutor.execute(new FourThreadPoolsRunnableImpl(index));
}


/*newCachedThreadPool.shutdown();
newFixedThreadPool.shutdown();
newScheduledThreadPool.shutdown();
newSingleThreadExecutor.shutdown();*/


//执行的任务类
public class FourThreadPoolsRunnableImpl implements Runnable{

    private Integer index;
    public FourThreadPoolsRunnableImpl(Integer index)
    {
        this.index=index;
    }
    @Override
    public void run() {
        try {
            System.out.println("开始处理线程,index="+index);
            Thread.sleep(3000);
            System.out.println("执行完毕,index="+index+" 线程标识:"+this.toString());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 实际项目使用

```java
//定义线程安全线程池
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
        resultFutures.put(key, ((ExecutorService)serviceRef.get()).submit((Callable)tasks.get(key)));
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
//第三部, 提供使用完成之后停止线程池的方法
public static void shutdown() {
    if (serviceRef.get() != null) {
        synchronized(ThreadPoolExecutorDemo.class) {
            if (serviceRef.get() != null) {
                ((ExecutorService)serviceRef.get()).shutdown();
                serviceRef.set(null);
            }
        }
    }
}
```


## 参考

[Java并发编程：线程池的使用](https://www.cnblogs.com/dolphin0520/p/3932921.html)

[四种Java线程池用法解析](https://www.cnblogs.com/ruiati/p/6134131.html)