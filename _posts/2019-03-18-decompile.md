---
layout: post
title:  "Java反编译"
categories: java-basics
tags:  java-basics
author: 网络
---

* content
{:toc}


## 前言


总结java基础知识

##  课程目录

* `.class`文件结构
* 反编译工具







## `.class`文件结构

https://blog.csdn.net/xingkongdeasi/article/details/79688505

https://dzone.com/articles/introduction-to-java-bytecode

## 工具

### javap反汇编class文件

`javap -help` 查看所有命令

常用命令：

```
javap -c <xxx.class> 反汇编代码逻辑
```

```
javap -v <xxx.class> 反汇编代码逻辑及附加逻辑
```

[Oracle官方文档-java字节码指令](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5)

[java字节码指令集详细介绍](https://www.cnblogs.com/vinozly/p/5399308.html)

[通过javap命令分析java汇编指令](https://www.jianshu.com/p/6a8997560b05)

### Java Decompiler反编译class文件

http://java-decompiler.github.io/

* jd-GUI
* jd-intellij
* jd-eclipse

## 参考

[JVM参数汇总](https://www.cnblogs.com/duanxz/p/3482366.html)