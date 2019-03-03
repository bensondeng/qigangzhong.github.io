---
layout: post
title:  "ReentrantLock"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}


## 前言

总结java线程基础知识

##  课程目录
* ReentrantLock
* ReentrantReadWriteLock







### `ReentrantLock`

#### 与synchronized的相同点

* 都是排它锁，性能差不多
* 都支持重入

#### 与synchronized的不同点

* ReentrantLock同一个锁加锁和解锁的次数必须一致，少解锁一次会导致其他线程一直在等待，造成死锁
* ReentrantLock支持公平锁/非公平锁，而synchronized只是非公平锁
* ReentrantLock可以响应中断
* ReentrantLock可以进行加锁超时设置
* ReentrantLock提供Condition来实现等待-通知机制

```java
//实现一个阻塞队列
public class MyBlockingQueue<E> {

    int size;//阻塞队列最大容量

    ReentrantLock lock = new ReentrantLock();

    LinkedList<E> list=new LinkedList<>();//队列底层实现

    Condition notFull = lock.newCondition();//队列满时的等待条件
    Condition notEmpty = lock.newCondition();//队列空时的等待条件

    public MyBlockingQueue(int size) {
        this.size = size;
    }

    public void enqueue(E e) throws InterruptedException {
        lock.lock();
        try {
            while (list.size() ==size)//队列已满,在notFull条件上等待
                notFull.await();
            list.add(e);//入队:加入链表末尾
            System.out.println("入队：" +e);
            notEmpty.signal(); //通知在notEmpty条件上等待的线程
        } finally {
            lock.unlock();
        }
    }

    public E dequeue() throws InterruptedException {
        E e;
        lock.lock();
        try {
            while (list.size() == 0)//队列为空,在notEmpty条件上等待
                notEmpty.await();
            e = list.removeFirst();//出队:移除链表首元素
            System.out.println("出队："+e);
            notFull.signal();//通知在notFull条件上等待的线程
            return e;
        } finally {
            lock.unlock();
        }
    }
}
```

### `ReaderLock WriterLock`

* 写锁通过重入的方式降级为读锁，反过来读锁不支持升级到写锁

```java
//写锁通过重入的方式降级为读锁，反过来读锁不支持升级到写锁
ReentrantReadWriteLock rwLock=new ReentrantReadWriteLock();
rwLock.writeLock().lock();
System.out.println("get write lock");
rwLock.readLock().lock();
System.out.println("get read lock");
rwLock.readLock().unlock();
System.out.println("read unlock");
```

* 读锁之间可以共享，同一把锁多个线程可以同时加锁。
* 写锁是独占的，写锁与写锁，写锁与读锁，都不能同时加锁

```java
public class ReadAndWriteLockTest {

    public static ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public static void main(String[] args) {

        //同时写
        ExecutorService service = Executors.newCachedThreadPool();
        service.execute(new Runnable() {
            @Override
            public void run() {
                writeFile(Thread.currentThread());
            }
        });
        service.execute(new Runnable() {
            @Override
            public void run() {
                writeFile(Thread.currentThread());
            }
        });

        service.shutdown();
    }

    // 读操作
    public static void readFile(Thread thread) {
        lock.readLock().lock();
        boolean isWriteLocked = lock.isWriteLocked();
        if (!isWriteLocked) {
            System.out.println("当前为读锁！");
        }
        try {
            for (int i = 0; i < 5; i++) {
                try {
                    Thread.sleep(20);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(thread.getName() + ":正在进行读操作……");
            }
            System.out.println(thread.getName() + ":读操作完毕！");
        } finally {
            System.out.println("释放读锁！");
            lock.readLock().unlock();
        }
    }

    // 写操作
    public static void writeFile(Thread thread) {
        lock.writeLock().lock();
        boolean isWriteLocked = lock.isWriteLocked();
        if (isWriteLocked) {
            System.out.println("当前为写锁！");
        }
        try {
            for (int i = 0; i < 5; i++) {
                try {
                    Thread.sleep(20);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(thread.getName() + ":正在进行写操作……");
            }
            System.out.println(thread.getName() + ":写操作完毕！");
        } finally {
            System.out.println("释放写锁！");
            lock.writeLock().unlock();
        }
    }
}
```

## 参考

[ReentrantLock(重入锁)功能详解和应用演示](https://www.cnblogs.com/takumicx/p/9338983.html)

[从源码角度彻底理解ReentrantLock(重入锁)](https://www.cnblogs.com/takumicx/p/9402021.html)

[ReadWriteLock读写锁的使用](https://www.jianshu.com/p/9cd5212c8841)