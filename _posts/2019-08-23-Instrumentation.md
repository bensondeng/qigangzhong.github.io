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

JDK自带的[java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)包其实也可以做到类似的事情，它也叫JavaAgent，或者java探针。它提供了两种方式来实现无侵入的instrument（监视？改造？增强？）你的java应用程序。可以结合`ASM`、`javassist`直接对字节码进行修改，让原有的功能更强大。

* premain(JDK5)

顾名思义是在你的应用main方法之前就执行一些操作，可以通过`java -javaagent:xxx.jar -jar xxx.jar`的方式在JVM启动的时候，在main方法执行之前做一些事情，可以进行一些监控操作，或者直接修改原有代码。

* agentmain(JDK6)

这个操作就更厉害了，可以在JVM启动之后attach到正在运行的JVM进程上，然后执行一些事情，监控原有代码，或者直接修改、替换字节码等等。

## premain

假设有一个自己的应用程序，里面有一个方法，我们需要在方法执行之前添加一些代码，打印这个方法的参数信息。

### 创建业务应用

应用名称为[instrument-demo-app](https://gitee.com/qigangzhong/java-basics/tree/master/instrument-demo/instrument-demo-app)，为了让这个简单的应用通过独立jar包可以运行，pom中添加maven-jar-plugin插件（在springboot应用中建议使用spring-boot-maven-plugin插件），使用maven的package命令直接打包成jar包instrument-demo-app-1.0.0.jar，并且我们使用maven插件来创建manifest文件一起打到jar包里面，相当于是在项目的`src/main/resources/META-INF/MANIFEST.MF`文件中添加一些属性。

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
                        <mainClass>com.qigang.instrument.demo.app.TestApplication</mainClass>
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
public class TestApplication {
    public static void main(String[] args) throws InterruptedException {
        TestObject object = new TestObject();
        for(;;){
            TimeUnit.SECONDS.sleep(2);
            object.testMethod("zhangsan", "zhangsan@123.com",30);
        }
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

### 创建premain探针应用

应用名称为[instrument-demo-premain](https://gitee.com/qigangzhong/java-basics/tree/master/instrument-demo/instrument-demo-premain)，结合javassist来修改instrument-demo-app-1.0.0.jar中原有的方法字节码，需要添加javassist依赖，以及打成独立jar包的maven插件，注意manifest的配置，可以参考[java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)中的说明。最终打成的探针jar包为instrument-demo-premain-1.0.0.jar。

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

java代码包括3个类：1. premain方法入口类，2. ClassFileTransformer类负责修改原有方法字节码注入日志代码行，3. 单独的日志打印类

```java
import java.lang.instrument.Instrumentation;
public class MyPreMainApplication extends Object {
    public static void premain(String agentArgs, Instrumentation inst) {
        //这里的addtransformer添加的ClassFileTransformer是针对所有的类的，每个类被加载的时候，MyClassFileTransformer.transform方法都会被执行一次
        //所以在MyClassFileTransformer.transform方法中要做好过滤，只处理需要修改的类
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

1.将instrument-demo-app打成jar包instrument-demo-app-1.0.0.jar，将探针应用instrument-demo-premain-1.0.0.jar打成jar包instrument-demo-premain-1.0.0.jar（依赖包在target/lib/目录下），为了测试方便，把2个jar包以及target/lib/目录全部copy到一起

2.启动应用instrument-demo-app-1.0.0.jar，通过-javaagent参数添加探针，可以看到日志打印代码已经被注入到方法中并执行

```bash
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

还是使用上面的例子，[instrument-demo-app](https://gitee.com/qigangzhong/java-basics/tree/master/instrument-demo/instrument-demo-app)应用的代码逻辑保持不变，我们现在先让instrument-demo-app-1.0.0.jar应用先跑起来，然后再将我们的探针应用jar包attach到跑起来的应用JVM进程上，在程序执行的过程中修改字节码。给奔跑中的汽车换轮子，很厉害的样子。

### 创建agentmain探针应用

应用名称为[instrument-demo-agentmain](https://gitee.com/qigangzhong/java-basics/tree/master/instrument-demo/instrument-demo-agentmain)，pom文件中的依赖以及maven插件与premain探针应用基本一样，除去manifest中的`Premain-Class`属性需要修改成`Agent-Class`。

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
                            <Agent-Class>com.qigang.instrument.demo.agentmain.MyAgentMainApplication</Agent-Class>
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

java代码也还是包括3个类：1. agentmain方法入口类，这个同premain探针入口方法有点区别，2. ClassFileTransformer子类不变，3. 单独的日志打印类也不变

```java
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class MyAgentMainApplication {
    public static void agentmain(String agentArgs, Instrumentation inst) {
        System.out.println("agentmain starting ****************************");

        inst.addTransformer(new MyClassFileTransformer(),true);

        try {
            //这一步很关键，探针被执行的时候需要主动转换需要改造的class
            inst.retransformClasses(Class.forName("com.qigang.instrument.demo.app.TestObject"));
        } catch (UnmodifiableClassException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
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

        System.out.println("transformer starting, className: "+className);

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
            //引入需要使用的class对应的包，这里引入的命名空间要正确
            ClassPool.getDefault().importPackage("com.qigang.instrument.demo.agentmain");
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

### attach探针应用到运行中的JVM应用进程上

想要将我们做好的agentmain探针应用生效，需要attach到运行中的JVM进程上，这需要用到JDK自带的tools.jar这个工具包，为了更好的隔离应用，我们创建一个单独的应用专门用来做attach这个事情，这个应用名称为[instrument-demo-agentmain-attach](https://gitee.com/qigangzhong/java-basics/tree/master/instrument-demo/instrument-demo-agentmain-attach)，它的代码比较简单，不需要任何配置文件和pom依赖：

```java
import com.sun.tools.attach.*;
import java.io.IOException;
import java.util.List;

public class MyAttachApplication {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor descriptor : list) {
            if (descriptor.displayName().endsWith("instrument-demo-app-1.0.0.jar")) {
                VirtualMachine virtualMachine = VirtualMachine.attach(descriptor.id());
                virtualMachine.loadAgent("E:\\agent\\instrument-demo-agentmain-1.0.0.jar", "arg1");
                virtualMachine.detach();
            }
        }
    }
}
```

tools.jar默认情况下不在classpath中，这个jar包在`${JAVA_HOME}/lib`目录下面，将tools.jar添加到IDEA的library中之后就可以正常运行（Project Structure-->Libraries-->Add）。

### 测试

1.将instrument-demo-app打成jar包instrument-demo-app-1.0.0.jar

2.将instrument-demo-agentmain打成jar包instrument-demo-agentmain-1.0.0.jar，依赖jar包都在target/lib目录下

3.为了测试方便，将2个jar包以及依赖包目录target/lib都copy到同一个目录（这里测试目录为E:\agent）下面

4.启动instrument-demo-app-1.0.0.jar，然后再执行instrument-demo-agentmain-attach进行探针包的attach动作（直接在idea中运行），会发现输出的结果发生了变化，代表探针修改字节码操作生效，参数日志信息也打印出来了

```bash
java -jar instrument-demo-app-1.0.0.jar

----------------------------------------------------------------------------
testMethod begin--------------------------------Mon Aug 26 17:01:08 CST 2019
name:zhangsan, email:zhangsan@123.com, age:30
testMethod end--------------------------------
testMethod begin--------------------------------Mon Aug 26 17:01:10 CST 2019
name:zhangsan, email:zhangsan@123.com, age:30
testMethod end--------------------------------
agentmain starting ****************************
transformer starting, className: com/qigang/instrument/demo/app/TestObject
transformer starting, className: com/qigang/instrument/demo/agentmain/MyLogger
this is insert log :zhangsan@123.com
this is insert log :zhangsan
testMethod begin--------------------------------Mon Aug 26 17:01:12 CST 2019
name:zhangsan, email:zhangsan@123.com, age:30
testMethod end--------------------------------
this is insert log :zhangsan@123.com
this is insert log :zhangsan
testMethod begin--------------------------------Mon Aug 26 17:01:14 CST 2019
name:zhangsan, email:zhangsan@123.com, age:30
testMethod end--------------------------------
----------------------------------------------------------------------------
```

### agentmain探针和attach应用打包在一起

上面的操作是agentmain探针应用打成jar包，attach应用测试时直接在idea中运行。其实可以将两者打成一个jar包，attach的JVM进程名称可以作为参数来指定，探针jar包的路径可以动态获取。具体可以查看[instrument-demo-agentmain](https://gitee.com/qigangzhong/java-basics/tree/master/instrument-demo/instrument-demo-agentmain)源码。

## 其它

其实instrument还有其它用途：

* 修改classpath

主要依赖`appendToBootstrapClassLoaderSearch`和`appendToSystemClassLoaderSearch`这两个方法，从名称可以判断出意思。

* 针对本地代码的instrumentation

较少使用，不介绍了。

## 参考

[java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

[JAVA instrument简单使用](https://blog.csdn.net/jzsxyj1111/article/details/89358541)

[*****动态代理方案性能对比(梁飞博客)](https://www.iteye.com/blog/javatar-814426)

[Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)

[使用maven-jar-plugin定制MANIFEST.MF文件](https://www.ibm.com/developerworks/cn/java/j-5things13/index.html)
