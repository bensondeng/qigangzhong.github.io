---
layout: post
title:  "Instrumentation"
categories: java-basics
tags:  java-basics
author: 网络
---

* content
{:toc}


## 前言


总结`java.lang.instrument.Instrumentation`相关基础知识

## 课程目录

* premain
* agentmain










## 介绍

java中可以通过动态代理的方式来做一些AOP的事情，例如使用`JDK Proxy`、`CGLIB`、`ASM`、`javassist`，在不修改源码的情况下在一个类的方法执行前后做一些事情。使用`ASM`和`javasist`包还可以直接修改编译后的class。

JDK自带的[java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)包其实也可以做到类似的事情，它也叫JavaAgent，或者java探针。它提供了两种方式来实现无侵入的instrument（监视？改造？）你的java应用程序。可以结合`ASM`、`javassist`直接对字节码进行修改，让原有的功能更强大。

* premain

顾名思义是在你的应用main方法之前就执行一些操作，可以通过`java -javaagent:xxx.jar -jar xxx.jar`的方式在JVM启动的时候，在main方法执行之前做一些事情，可以进行一些监控操作，或者直接修改原有代码。

* agentmain

这个操作就更厉害了，可以在JVM启动之后attach到正在运行的JVM进程上，然后执行一些事情。

## premain

假设有一个自己的应用程序，里面有一个方法，我们需要在方法执行之前添加一些代码，打印这个方法的参数信息。

### 创建普通应用

为了让这个简单的应用通过独立jar包可以运行，添加maven插件，使用maven的package命令直接打包成jar包

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.1.0</version>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>com.qigang.instrument.demo.app.Application</mainClass>
                        <addClasspath>true</addClasspath>
                        <classpathPrefix>lib/</classpathPrefix>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```java
public class Application {
    public static void main(String[] args) {
        TestObject object = new TestObject();
        object.testMethod("zhangsan", "zhangsan@123.com",30);
    }
}

public interface TestInterface {
    void testMethod(String name, String email, Integer age);
}

public class TestObject implements TestInterface {
    @Override
    public void testMethod(String name, String email, Integer age) {
        System.out.println("testMethod begin");
        System.out.println(String.format("name:%s, email:%s, age:%s", name,email,age));
        System.out.println("testMethod end");
    }
}
```

### 创建agent应用

结合javassist来修改原有的方法，所以需要添加一些依赖，以及打成独立jar包的maven插件，注意manifest的配置，可以参考[java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)中的说明。

```xml
<dependencies>
    <!--使用javassit修改字节码-->
    <dependency>
        <groupId>org.javassist</groupId>
        <artifactId>javassist</artifactId>
        <version>3.23.1-GA</version>
    </dependency>
    <!--工具包-->
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.5</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.1.0</version>
            <configuration>
                <archive>
                    <manifest>
                        <addClasspath>true</addClasspath>
                        <classpathPrefix>lib/</classpathPrefix>
                    </manifest>
                    <manifestEntries>
                        <Premain-Class>com.qigang.instrument.demo.premain.MyPreMainApplication</Premain-Class>
                        <Can-Redefine-Classes>true</Can-Redefine-Classes>
                        <Can-Retransform-Classes>true</Can-Retransform-Classes>
                    </manifestEntries>
                </archive>
            </configuration>
        </plugin>

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy</id>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/lib</outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

java代码包括3个类：premain方法入口类，ClassFileTransformer类负责修改原有方法字节码注入日志代码行，以及日志打印类

```java
import java.lang.instrument.Instrumentation;
public class MyPreMainApplication extends Object {
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new MyClassFileTransformer());
    }
}

import javassist.ClassPool;
import javassist.CtBehavior;
import javassist.CtClass;
import org.apache.commons.lang3.StringUtils;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.HashSet;
import java.util.Set;

public class MyClassFileTransformer implements ClassFileTransformer {

    private static Set<String> interFaceList = new HashSet<>();

    static {
        interFaceList.add("com.qigang.instrument.demo.app.TestInterface");
    }

    /**
     * transform方法会处理所有加载的类
     * @param loader
     * @param className
     * @param classBeingRedefined
     * @param protectionDomain
     * @param classfileBuffer
     * @return
     * @throws IllegalClassFormatException
     */
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        try {
            if (StringUtils.isBlank(className)) {
                return null;
            }
            String currentClassName = className.replaceAll("/", ".");
            CtClass currentClass = ClassPool.getDefault().get(currentClassName);
            CtClass[] interfaces = currentClass.getInterfaces();
            if (!needProcess(interfaces)) {
                return null;
            }
            //引入需要使用的class对应的包
            ClassPool.getDefault().importPackage("com.qigang.instrument.demo.premain");
            CtBehavior[] methods = currentClass.getMethods();
            for (CtBehavior method : methods) {
                String methodName = method.getName();
                if ("testMethod".equals(methodName)) {
                    CtClass[] paramTypes = method.getParameterTypes();
                    for(int i=0;i<paramTypes.length;i++){
                        CtClass paramType = paramTypes[i];
                        String typeName = paramType.getName();
                        if ((String.class.getName().replaceAll("/", ".")).equals(typeName)) {
                            //如果方法参数类型为字符串，则打印该参数值，下标从1开始
                            method.insertAt(0, " MyLogger.log($"+(i+1)+");");
                        }
                    }
                }
            }
            return currentClass.toBytecode();

        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return null;
    }

    private boolean needProcess(CtClass[] interfaces) {
        for (int i = 0; i < interfaces.length; i++) {
            if (interFaceList.contains(interfaces[i].getName())) {
                return true;
            }
        }
        return false;
    }
}

public class MyLogger {
    public static void log(String message) {
        System.out.println("this is insert log :" + message);
    }
}
```

### 测试

```bash
# 1. 将普通的java应用，以及探针引用分别打成独立的jar包，并放到一个目录下（依赖包./lib/*.jar也一起copy过去）
mvn package

# 2. 启动应用，可以看到日志打印代码已经被注入到方法中并执行
java -javaagent:instrument-demo-premain-1.0.0.jar -jar instrument-demo-app-1.0.0.jar
----------------------------------------------
this is insert log :zhangsan@123.com
this is insert log :zhangsan
testMethod begin
name:zhangsan, email:zhangsan@123.com, age:30
testMethod end
----------------------------------------------
```

## agentmain

## 参考

[JAVA instrument简单使用](https://blog.csdn.net/jzsxyj1111/article/details/89358541)

[*****动态代理方案性能对比(梁飞博客)](https://www.iteye.com/blog/javatar-814426)
