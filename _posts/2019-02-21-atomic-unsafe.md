---
layout: post
title:  "atomic、unsafe"
categories: thread
tags:  thread
author: 网络
---

* content
{:toc}

总结java线程基础知识

* atomic、unsafe









## 一. 基本概念

### 1. CAS及ABA

> * CAS原理
>
> CAS(compare and swap)包含3个参数CAS(V,E,N).V表示要更新的变量, E表示预期值, N表示新值.  
> 仅当V值等于E值时, 才会将V的值设为N, 如果V值和E值不同, 则说明已经有其他线程做了更新, 则当前线程什么都不做. 最后, CAS返回当前V的真实值. CAS操作是抱着乐观的态度进行的, 它总是认为自己可以成功完成操作.当多个线程同时使用CAS操作一个变量时, 只有一个会胜出, 并成功更新, 其余均会失败.失败的线程不会被挂起,仅是被告知失败, 并且允许再次尝试, 当然也允许失败的线程放弃操作.基于这样的原理, CAS操作即时没有锁,也可以发现其他线程对当前线程的干扰, 并进行恰当的处理.
>
> * ABA问题
>
> 线程一准备用CAS将变量的值由A替换为B, 在此之前线程二将变量的值由A替换为C, 线程三又将C替换为A, 然后线程一执行CAS时发现变量的值仍然为A, 所以线程一CAS成功

### 2. Unsafe

java不能直接访问操作系统底层，通过本地方法来实现，Unsafe类提供了硬件级别的原子操作，CAS相关的原子操作类AtomicXXX都是通过Unsafe类来实现的

Unsafe的作用:
* 分配内存/释放内存

```java
public native long allocateMemory(long l); //分配内存
public native long reallocateMemory(long l, long l1); //扩充内存
public native void freeMemory(long l); //释放内存
```

* 定位对象某字段的内存位置/修改对象字段值

```java
public final class Unsafe {
    public static final int ARRAY_INT_BASE_OFFSET;
    public static final int ARRAY_INT_INDEX_SCALE;

    /*
        一个java对象可以看成是一段内存，各个字段都得按照一定的顺序放在这段内存里，同时考虑到对齐要求，可能这些字段不是连续放置的，用这个方法能准确地告诉你某个字段相对于对象的起始内存地址的字节偏移量，因为是相对偏移量，所以它其实跟某个具体对象又没什么太大关系，跟class的定义和虚拟机的内存模型的实现细节更相关。
    */
    public native long staticFieldOffset(Field field);
    public native long objectFieldOffset(Field var1);
    public native int getIntVolatile(Object obj, long l);
    public native long getLong(Object obj, long l);
    public native int arrayBaseOffset(Class class1);
    public native int arrayIndexScale(Class class1);

    static 
    {
        ARRAY_INT_BASE_OFFSET = theUnsafe.arrayBaseOffset(int[].class);
        ARRAY_INT_INDEX_SCALE = theUnsafe.arrayIndexScale(int[].class);
    }
}
```

```java
//通过objectFieldOffset实现的sizeOf方法，计算对象大小
public class UnsafeDemo {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        String input="hello world";
        System.out.println(sizeOf(input));
    }

    public static Unsafe getUnsafe() throws IllegalAccessException, NoSuchFieldException {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);
        return unsafe;
    }

    public static long sizeOf(Object o) throws NoSuchFieldException, IllegalAccessException {
        Unsafe u = getUnsafe();
        HashSet<Field> fields = new HashSet<>();
        Class c = o.getClass();
        while (c != Object.class) {
            for (Field f : c.getDeclaredFields()) {
                if ((f.getModifiers() & Modifier.STATIC) == 0) {
                    fields.add(f);
                }
            }
            c = c.getSuperclass();
        }

        // get offset
        long maxSize = 0;
        for (Field f : fields) {
            long offset = u.objectFieldOffset(f);
            if (offset > maxSize) {
                maxSize = offset;
            }
        }

        return ((maxSize/8) + 1) * 8;   // padding
    }
}
```

* 线程挂起/恢复

```java
public native void unpark(Object var1);
public native void park(boolean var1, long var2);
```


### 3. 原子操作相关类

> 原子类型位于java.util.concurrent.atomic包下，主要分为4种类型：
> 1. 普通原子类型：提供对boolean、int、long和对象的原子性操作
> > AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference
> 2. 原子类型数组：提供对数组元素的原子性操作
> > AtomicIntegerArray，AtomicLongArray，AtomicReferenceArray
> 3. 原子类型字段更新器：提供对指定对象的指定字段进行原子性操作
> > AtomicLongFieldUpdater，AtomicIntegerFieldUpdater，AtomicReferenceFieldUpdater
> 4. 带版本号的原子引用类型：以版本戳的方式解决原子类型的ABA问题
> > AtomicMarkableReference，AtomicStampedReference
> 5. 原子累加器(JDK1.8)：AtomicLong和AtomicDouble的升级类型，专门用于数据统计，性能更高
> > DoubleAccumulator，DoubleAdder，LongAccumulator，LongAdder

### 3. 示例

#### AtomicReference示例

```java
import java.util.concurrent.atomic.AtomicReference;

public class AtomicReferenceTest {
    public final static AtomicReference<String> attxnicStr = new AtomicReference<String>("abc");

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread() {
                public void run() {
                    try {
                        Thread.sleep(Math.abs((int) (Math.random() * 100)));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    if (attxnicStr.compareAndSet("abc", "def")) {
                        System.out.println("Thread:" + Thread.currentThread().getId() + " change value to " + attxnicStr.get());
                    } else {
                        System.out.println("Thread:" + Thread.currentThread().getId() + " change failed!");
                    }
                }
            }.start();
        }
    }
}
```

#### AtomicStampedReference示例

```java
public class AtomicStampedReferenceDemo {
 static AtomicStampedReference<Integer> money=new AtomicStampedReference<Integer>(19,0);
    public staticvoid main(String[] args) {
        //模拟多个线程同时更新后台数据库，为用户充值
        for(int i = 0 ; i < 3 ; i++) {
            final int timestamp=money.getStamp();
            new Thread() {  
                public void run() { 
                    while(true){
                       while(true){
                           Integerm=money.getReference();
                            if(m<20){
                         if(money.compareAndSet(m,m+20,timestamp,timestamp+1)){
          　　　　　　　　　　　　　　　 System.out.println("余额小于20元，充值成功，余额:"+money.getReference()+"元");
                                    break;
                                }
                            }else{
                               //System.out.println("余额大于20元，无需充值");
                                break ;
                             }
                       }
                    }
                } 
            }.start();
         }
        
       //用户消费线程，模拟消费行为
        new Thread() { 
             publicvoid run() { 
                for(int i=0;i<100;i++){
                   while(true){
                        int timestamp=money.getStamp();
                        Integer m=money.getReference();
                        if(m>10){
                             System.out.println("大于10元");
                          　　if(money.compareAndSet(m, m-10,timestamp,timestamp+1)){
                      　　　　　　 System.out.println("成功消费10元，余额:"+money.getReference());
                                 break;
                             }
                        }else{
                           System.out.println("没有足够的金额");
                             break;
                        }
                    }
                    try {Thread.sleep(100);} catch (InterruptedException e) {}
                 }
            } 
        }.start(); 
    }
 }
```

#### AtomicIntegerArray示例

```java
import java.util.concurrent.atomic.AtomicIntegerArray;

public class AtomicArrayDemo {
    static AtomicIntegerArray arr = new AtomicIntegerArray(10);

    public static class AddThread implements Runnable {
        public void run() {
            for (int k = 0; k < 10000; k++) {
                //将指定下标的元素加1
                arr.incrementAndGet(k % arr.length());
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] ts = new Thread[10];
        for (int k = 0; k < 10; k++) {
            ts[k] = new Thread(new AddThread());
        }
        for (int k = 0; k < 10; k++) {
            ts[k].start();
        }
        for (int k = 0; k < 10; k++) {
            ts[k].join();
        }
        System.out.println(arr);
    }
}
```

#### AtomicIntegerFieldUpdater示例

```java
package automic;
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;
/**
 * 原子整型字段更新操作
 */
public class AtomicIntegerFieldUpdaterTest {
     private static Class<Person> cls;
     /**
      * AtomicIntegerFieldUpdater说明
      * 基于反射的实用工具，可以对指定类的指定 volatile int 字段进行原子更新。
      * @param args
      */
     public static void main(String[] args) {
        // 新建AtomicLongFieldUpdater对象，传递参数是“class对象”和“long类型在类中对应的名称”
        AtomicIntegerFieldUpdater<Person> mAtoLong = AtomicIntegerFieldUpdater.newUpdater(Person.class, "id");
        Person person = new Person(12345);
        mAtoLong.compareAndSet(person, 12345, 1000);
        System.out.println("id="+person.getId());
     }
}


package automic;
class Person {
    volatile int id;
    public Person(int id) {
        this.id = id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public int getId() {
        return id;
    }
}
```

> 1. 在上述的例子中，对于字段ID的修改，其中id的修饰必须是基本类型数据，用volatile修饰，不能是包装类型，int,long就可以，但是不可以是Integer和Long；
> 2. 必须是实例变量，不可以是类变量
> 3. 必须是可变的变量，不能是final修饰的变量
> 4. 不支持static字段
> 5. 只能修改可见字段，比如private字段无法修改

## 参考
[JAVA并发编程学习笔记之Unsafe类](https://blog.csdn.net/aesop_wubo/article/details/7537278)

[Java并发27:Atomic系列-原子类型累加器XxxxAdder和XxxxAccumulator的学习笔记](https://blog.csdn.net/hanchao5272/article/details/79689366)

[Java高并发之无锁与Atomic源码分析](https://www.cnblogs.com/xdecode/p/9022525.html)

[并发之AtomicIntegerFieldUpdater](https://www.cnblogs.com/gosaint/p/9064048.html)

[*****Java 里面有sizeof，计算内存占用的方法么](https://www.jianshu.com/p/b925b5b5610e?from=timeline&isappinstalled=0)

[github开源项目: ehcache-sizeof](https://github.com/ehcache/sizeof)