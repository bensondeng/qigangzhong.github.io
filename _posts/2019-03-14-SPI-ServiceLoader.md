---
layout: post
title:  "SPI-ServieLoader"
categories: arch
tags:  arch
author: 刚子
---

* content
{:toc}


## 前言

总结java基础知识

##  课程目录

* SPI
* ServiceLoader









## SPI是什么

SPI(Service Provider Interface)是JDK内置的一种服务提供发现机制。一个服务(Service)通常指的是已知的接口或者抽象类，服务提供方(Service Provider)就是对这个接口或者抽象类的实现，按照SPI标准在资源路径META-INF/services目录下创建一个以服务的全名称命名的文件，文件中配置服务提供方，类加载器就可以动态地加载服务提供方。通过这个配置文件来实现服务、服务提供方的解耦，生产服务的一方即为制定服务标准的一方，而服务提供方可以认为是实现标准的一方。

在JDK中可以通过java.util.ServiceLoader类来实现SPI机制。

## ServiceLoader

简单示例

```java
/**
 * Service
 */
public interface HelloService {
    void say();
}

/**
 * Service Provider 1
 */
public class Jack implements HelloService {
    @Override
    public void say() {
        System.out.println("hello, my name is jack...");
    }
}

/**
 * Service Provider 2
 */
public class John implements HelloService {
    @Override
    public void say() {
        System.out.println("hello, my name is john...");
    }
}

/**
在resources下新建/META-INF/services目录，新目录下创建以服务名称命名的文件com.qigang.spidemo.HelloService
文件内容如下:
com.qigang.spidemo.Jack
com.qigang.spidemo.John
*/

/**
 * 测试类
 */
public class Test {
    public static void main(String[] args) {
        ServiceLoader<HelloService> services = ServiceLoader.load(HelloService.class);
        for (HelloService helloService :services){
            helloService.say();
        }
    }
}

/**
结果：
hello, my name is jack...
hello, my name is john...
*/
```

如果你是一个标准服务方提供了一个服务但是想把服务提供方留给第三方来实现，那么通过SPI机制，你就可以在你的标准服务中调用第三方服务提供者的实现。JDK中的数据库驱动标准就是利用了SPI的机制，java.sql.Driver就是JDK提供的一个服务，DriverManager类是JDK内置的一个管理类，它内部就是通过SPI机制来获取服务提供方来创建数据库连接，mysql、sqlserver等都有对应的服务提供方具体实现。

```java
package java.sql;
public class DriverManager {
    //...
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }

    private static void loadInitialDrivers() {
        String drivers;
        try {
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty("jdbc.drivers");
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }
        // If the driver is packaged as a Service Provider, load it.
        // Get all the drivers through the classloader
        // exposed as a java.sql.Driver.class service.
        // ServiceLoader.load() replaces the sun.misc.Providers()

        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                /* Load these drivers, so that they can be instantiated.
                 * It may be the case that the driver class may not be there
                 * i.e. there may be a packaged driver with the service class
                 * as implementation of java.sql.Driver but the actual class
                 * may be missing. In that case a java.util.ServiceConfigurationError
                 * will be thrown at runtime by the VM trying to locate
                 * and load the service.
                 *
                 * Adding a try catch block to catch those runtime errors
                 * if driver not available in classpath but it's
                 * packaged as service and that service is there in classpath.
                 */
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

        println("DriverManager.initialize: jdbc.drivers = " + drivers);

        if (drivers == null || drivers.equals("")) {
            return;
        }
        String[] driversList = drivers.split(":");
        println("number of Drivers:" + driversList.length);
        for (String aDriver : driversList) {
            try {
                println("DriverManager.Initialize: loading " + aDriver);
                Class.forName(aDriver, true,
                        ClassLoader.getSystemClassLoader());
            } catch (Exception ex) {
                println("DriverManager.Initialize: load failed: " + ex);
            }
        }
    }

    //...
}
```

## ServiceLoader-ClassLoader

通过查看`ServiceLoader.load(Class<S> service)`方法可以看到其内部是通过线程上下文类加载器`Thread.currentThread().getContextClassLoader()`来加载服务提供者的，只能加载classpath下的类，也就是说服务提供者实现以及那个配置文件必须放在程序运行的classpath下才行，如果我们想加载一个网络上的服务提供者，或者其他处于classpath之外的一个服务提供者，就需要使用到另外一个重载方法`ServiceLoader.load(Class<S> service, ClassLoader loader)`，通过一个自定义的ClassLoader可以实现该目的。

简单示例：

```java
ClassLoader cl = new URLClassLoader(
    new URL[] { new URL("file:" + "D:/Jack.jar"), new URL("file:" + "D:/John.jar") },
    SPIDemo.class.getClassLoader().getParent());
ServiceLoader<HelloService> services = ServiceLoader.load(HelloService.class, cl);
```

## google AutoService

每次新增加一个服务提供方就必须在/META-INF/services的服务配置文件中添加一条服务提供方的全名称，这样做很麻烦且容易遗忘，通过google AutoService可以利用java中的annotation来编译期间自动生成这个配置文件。

```xml
<dependency>
    <groupId>com.google.auto.service</groupId>
    <artifactId>auto-service</artifactId>
    <version>1.0-rc4</version>
</dependency>
```

```java
//利用google AutoService组件，编译时自动生成/META-INF/services下的文件
@AutoService(HelloService.class)
public class Jack implements HelloService {
    @Override
    public void say() {
        System.out.println("hello, my name is jack...");
    }
}
```

## 参考

[AutoService](https://github.com/google/auto/tree/master/service)
