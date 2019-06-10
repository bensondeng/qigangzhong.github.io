---
layout: post
title:  "ForkJoinPool"
categories: thread
tags:  thread ForkJoinPool
author: 网络
---

* content
{:toc}









## 一、 概念

![ForkJoinPool.jpg](/images/thread/ForkJoinPool.jpg) ![ThreadPoolExecutor.jpg](/images/thread/ThreadPoolExecutor.jpg)

ForkJoinPool与ThreadPoolExecutor都继承于AbstractExecutorService线程池抽象类，所以都是线程池的一种。ForkJoinPool利用Fork/Join分治思想，将一个大的任务拆成小任务，小任务再继续拆成更小的任务，一直拆到不能再拆（最小任务的粒度达到一个阀值），然后再递归汇总每个任务计算的值，得到最终的结果。

```
if(任务足够小）{
    直接计算得到结果
}else{
    分拆成N个子任务
    调用子任务的fork()进行计算
    调用子任务的join()合并计算结果
}
```

ForkJoinPool区别于普通ThreadPoolExecutor线程池的地方在于它提供的`work-stealing`算法。一个ForkJoinPool中存在多个工作线程`ForkJoinWorkerThread`，一个工作线程有自己的任务队列`WorkQueue（双端队列）`，队列中保存的是待执行的任务`ForkJoinTask`。一个工作线程从自身队列的**顶端**获取任务处理，当处理完所有的任务的时候它可以从其他工作线程的队列的**尾端**获取任务来处理。

ForkJoinTask是个抽象类，具体的实现有2个，RecursiveAction是无执行返回结果的，RecursiveTask是可以返回执行结果的。我们实现自己的处理任务只要继承这2个类就可以了。

> ForkJoinTask保存了它所属的ForkJoinPool以及WorkQueue的信息，任务执行时可以获取线程池的大小，队列大小等信息

![RecursiveAction.jpg](/images/thread/RecursiveAction.jpg) ![RecursiveTask.jpg](/images/thread/RecursiveTask.jpg)

可以通过new的方式来初始化ForkJoinPool线程池，指定线程数等参数。也可以通过ForkJoinPool.commonPool()初始化一个线程池，该静态方法默认使用CPU的核心数作为线程池中的线程数，可以满足大部分使用场景，推荐使用。CompletableFuture的实现中默认就是使用ForkJoinPool.commonPool()创建出来的线程池。

## 二、示例

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

/**
 * 模拟递归处理列表中的数据
 */
public class RecursiveActionDemo1 {
    class SendMsgTask extends RecursiveAction {
        private final int THRESHOLD = 10;
        private int start;
        private int end;
        private List<String> list;

        public SendMsgTask(int start, int end, List<String> list) {
            this.start = start;
            this.end = end;
            this.list = list;
        }

        @Override
        protected void compute() {
            if ((end - start) <= THRESHOLD) {
                for (int i = start; i < end; i++) {
                    System.out.println(Thread.currentThread().getName() + ": " + list.get(i));
                }
            }else {
                int middle = (start + end) / 2;
                invokeAll(new SendMsgTask(start, middle, list), new SendMsgTask(middle, end, list));
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        List<String> list = new ArrayList<>();
        for (int i = 0; i < 123; i++) {
            list.add(String.valueOf(i+1));
        }

        ForkJoinPool pool = new ForkJoinPool();
        pool.submit(new RecursiveActionDemo1().new SendMsgTask(0, list.size(), list));
        pool.awaitTermination(10, TimeUnit.SECONDS);
        pool.shutdown();
    }
}
```

```java
import java.util.Arrays;
import java.util.concurrent.*;

/**
 * 模拟对列表里面的数据进行排序
 */
public class RecursiveActionDemo2 {
    private static class SortTask extends RecursiveAction {
        static final int THRESHOLD = 100;
        final long[] array;
        final int lo, hi;

        public SortTask(long[] array, int lo, int hi) {
            this.array = array;
            this.lo = lo;
            this.hi = hi;
        }

        public SortTask(long[] array) {
            this(array, 0, array.length);
        }

        public void sortSequentially(int lo, int hi) {
            Arrays.sort(array, lo, hi);
        }

        public void merge(int lo, int mid, int hi) {
            long[] buf = Arrays.copyOfRange(array, lo, mid);
            for (int i = 0, j = lo, k = mid; i < buf.length; j++) {
                array[j] = (k == hi || buf[i] < array[k]) ? buf[i++] : array[k++];
            }
        }

        @Override
        protected void compute() {
            if (hi - lo < THRESHOLD) {
                sortSequentially(lo, hi);
            }else {
                int mid = (lo + hi) >>> 1;
                invokeAll(new SortTask(array, lo, mid), new SortTask(array, mid, hi));
                merge(lo, mid, hi);
            }
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long[] array = new long[120];
        for (int i = 0; i < array.length; i++) {
            array[i] = (long) (Math.random() * 1000);
        }
        System.out.println(Arrays.toString(array));

        ForkJoinPool pool = new ForkJoinPool();
        pool.submit(new SortTask(array));
        pool.awaitTermination(5, TimeUnit.SECONDS);
        pool.shutdown();

        for(int i=0;i<array.length;i++){
            System.out.println(array[i]);
        }
    }
}
```

```java
import java.util.concurrent.*;

/**
 * 模拟列表中的所有数字进行相加
 */
public class RecursiveTaskDemo1 {
    private class SumTask extends RecursiveTask<Integer> {
        private static final int THRESHOLD = 20;
        private int arr[];
        private int start;
        private int end;

        public SumTask(int[] arr, int start, int end) {
            this.arr = arr;
            this.start = start;
            this.end = end;
        }

        private Integer subtotal() {
            Integer sum = 0;
            for (int i = start; i < end; i++) {
                sum += arr[i];
            }
            System.out.println(Thread.currentThread().getName() + ": ∑(" + start + "~" + end + ")=" + sum);
            return sum;
        }

        @Override
        protected Integer compute() {
            if ((end - start) <= THRESHOLD) {
                return subtotal();
            }else {
                int middle = (start + end) / 2;
                SumTask left = new SumTask(arr, start, middle);
                SumTask right = new SumTask(arr, middle, end);
                left.fork();
                right.fork();

                return left.join() + right.join();
            }
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        int[] arr = new int[100];
        for (int i = 0; i < 100; i++) {
            arr[i] = i + 1;
        }

        ForkJoinPool pool = new ForkJoinPool();
        ForkJoinTask<Integer> result = pool.submit(new RecursiveTaskDemo1().new SumTask(arr, 0, arr.length));
        System.out.println("最终计算结果: " + result.invoke());
        pool.shutdown();
    }
}
```

```java
import java.util.concurrent.*;

/**
 * 模拟计算斐波那契序列的第n个值
 */
public class RecursiveTaskDemo2 {
    private static class Fibonacci extends RecursiveTask<Integer> {
        final int n;
        public Fibonacci(int n) {
            this.n = n;
        }

        @Override
        protected Integer compute() {
            if (n <= 2) {
                return 1;
            }else {
                Fibonacci f1 = new Fibonacci(n - 1);
                f1.fork();
                Fibonacci f2 = new Fibonacci(n - 2);
                f2.fork();
                return f1.join()+f2.join();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ForkJoinPool pool = new ForkJoinPool();
        Future<Integer> future = pool.submit(new Fibonacci(10));
        System.out.println(future.get());
        pool.shutdown();
    }
}
```

## 参考

[Java Fork/Join 框架](https://www.cnblogs.com/cjsblog/p/9078341.html)

[分析jdk-1.8-ForkJoinPool实现原理](https://www.jianshu.com/p/de025df55363)
