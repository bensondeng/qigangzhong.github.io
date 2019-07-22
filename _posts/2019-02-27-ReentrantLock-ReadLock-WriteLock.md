---
layout: post
title:  "ReentrantLock"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}

总结java线程基础知识

* ReentrantLock
* ReentrantReadWriteLock







### ReentrantLock

#### 与synchronized的相同点

* 都是排它锁，性能差不多
* 都支持重入

#### 与synchronized的不同点

* ReentrantLock同一个锁加锁和解锁的次数必须一致，少解锁一次会导致其他线程一直在等待，造成死锁
* ReentrantLock支持公平锁/非公平锁，而synchronized只是非公平锁
* ReentrantLock可以响应中断
* ReentrantLock可以进行加锁超时设置
* ReentrantLock提供Condition来实现等待-通知机制，用来替换Object.waite()，Object.nofity()方法
  * [Condition.await()](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/Condition.html#await())方法调用时会释放当前锁，当前线程暂停

[示例代码](https://gitee.com/qigangzhong/java-basics/tree/master/threads/src/main/java/com/qigang/async/lockdemo/blockingqueuedemo)：利用Condition来创建自定义的阻塞队列

```java
//实现一个阻塞队列
import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;
public class MyBlockingQueue<E> {

    //阻塞队列最大容量
    int size;

    ReentrantLock lock = new ReentrantLock();

    //队列底层实现
    LinkedList<E> list=new LinkedList<>();

    //队列满时的等待条件
    Condition notFull = lock.newCondition();
    //队列空时的等待条件
    Condition notEmpty = lock.newCondition();

    public MyBlockingQueue(int size) {
        this.size = size;
    }

    public void enqueue(E e) throws InterruptedException {
        try {
            lock.lock();

            String threadName = Thread.currentThread().getName();
            System.out.println(threadName+" 开始生产，锁的hashCode: "+lock.hashCode());

            //队列已满,在notFull条件上等待
            while (list.size() ==size)
            {
                notFull.await();
            }
            //入队:加入链表末尾
            list.add(e);
            System.out.println("入队：" +e);
            //通知在notEmpty条件上等待的线程
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public E dequeue() throws InterruptedException {
        try {
            lock.lock();
            String threadName = Thread.currentThread().getName();
            System.out.println(threadName+" 开始消费，锁的hashCode: "+lock.hashCode());

            //队列为空,在notEmpty条件上等待
            while (list.size() == 0)
            {
                notEmpty.await();
            }
            //出队:移除链表首元素
            E e = list.removeFirst();

            System.out.println(threadName + " 出队："+e);
            //通知在notFull条件上等待的线程
            notFull.signal();
            return e;
        } finally {
            lock.unlock();
        }
    }
}
```

### [ReetrantReadWriteLock](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)

读写锁的机制：
* "读-读" 不互斥
* "读-写" 互斥
* "写-写" 互斥

锁的升级、降级：

写锁可以降级为读锁，反过来读锁不可以升级为写锁。就是说同一个线程获取写锁之后还可以再次获取读锁，不会死锁，但是反过来不行。

```java
//同一个线程写锁通过重入的方式可以降级为读锁，反过来读锁不支持升级到写锁
ReentrantReadWriteLock rwLock=new ReentrantReadWriteLock();
rwLock.writeLock().lock();
System.out.println("get write lock");
rwLock.readLock().lock();
System.out.println("get read lock");
rwLock.readLock().unlock();
System.out.println("read unlock");
rwLock.writeLock().unlock();
System.out.println("write unlock");

System.out.println(rwLock.getReadHoldCount());
System.out.println(rwLock.getWriteHoldCount());
```

读写缓存的示例，锁降级的目的是为了提高执行效率：

```java
public class CacheDemo {
    private Map<String, Object> map = new HashMap<>(128);
    private ReadWriteLock rwl = new ReentrantReadWriteLock();

    public Object get(String id){
        Object value = null;
        //首先开启读锁，开始读缓存数据
        rwl.readLock().lock();
        try{
            //如果缓存中没有释放读锁，上写锁
            if(map.get(id) == null){
                rwl.readLock().unlock();
                rwl.writeLock().lock();
                try{
                    //防止多写线程重复查询赋值
                    if(map.get(id) == null){
                        //此时可以去数据库中查找，这里简单的模拟一下
                        value = "value from thread: "+Thread.currentThread().getName();
                        map.put(id,value);
                    }

                    //写完成之后立即加读锁进行降级,目的是让写操作之后的逻辑不会影响其他线程的读操作（提高效率），但读锁这个时候还没有释放，可以继续执行读操作或者其它逻辑
                    rwl.readLock().lock();
                }finally{
                    //写操作完成之后立即降级为读锁，然后释放写锁
                    rwl.writeLock().unlock();
                }
            }

            //进行读操作
            return map.get(id);
        }finally{
            rwl.readLock().unlock(); //最终释放读锁
        }
    }
}
```

读锁之间可以共享，同一把锁多个线程可以同时加锁。写锁是独占的，写锁与写锁，写锁与读锁，都不能同时加锁。

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

[*****Java多线程之ReentrantLock与Condition](https://www.cnblogs.com/xiaoxi/p/7651360.html)
