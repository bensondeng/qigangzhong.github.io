---
layout: post
title:  "springcloud-zipkin(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 介绍

zipkin的功能就是分布式链路追踪，基于[Dapper理论](http://bigbully.github.io/Dapper-translation/)，系统中微服务比较多的时候可以根据链路来判断请求经过的服务，调用时长，有无报错。zipkin对服务业务代码无侵入，直接引入pom依赖，添加配置即可实现追踪数据收集。追踪数据默认存储在zipkin-server的内存，可以更换为ES（推荐），MySql，RabbitMQ等。

其它链路追踪的产品有[CAT](https://github.com/dianping/cat)，[SkyWalking](https://github.com/apache/skywalking)，[PinPoint](https://github.com/naver/pinpoint)等，相对来说zipkin功能比较单一，并且对定能影响较大，生产环境建议选择SkyWalking或CAT这种生产级别的组件。

这里仅使用默认的内存存储的方式来演示，对于zipkin+rabbitmq+es的解决方案可以google一下。

## zipkin-server

目前zipkin官方不建议自行搭建zipkin-server站点，而是使用[zipkin站点独立jar包](https://dl.bintray.com/openzipkin/maven/io/zipkin/java/zipkin-server/)，下载之后通过`java -jar`的方式启动（注意是带-exec的jar包）。启动后默认端口9411，访问`http://localhost:9411`来进入UI界面。

```bash
java -jar zipkin-server-2.11.8-exec.jar
```

## 微服务集成sleuth+zipkin

springcloud sleuth组件集成了zipkin组件，在微服务项目中不需要更改任何业务代码，直接引入pom依赖，添加配置即可

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zipkin</artifactId>
    </dependency>
</dependencies>
```

```bash
# 【zipkin配置】
# zipkin服务器地址
spring.zipkin.base-url=http://localhost:9411
# 采样频率设置，1代表100%，默认0.1
spring.sleuth.sampler.probability=1
```

### 测试

1.启动eureka注册中心springcloud.f.eureka.server

2.启动zipkin-server这个jar包

3.启动两个测试服务springcloud.f.zipkin.service-hi，springcloud.f.zipkin.service-hi-client，后者通过feign调用前者

4.访问springcloud.f.zipkin.service-hi-client服务接口，观察zipkin-server上的链路数据

http://localhost:8862/callHiService

http://localhost:9411

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[openzipkin官方github](https://github.com/openzipkin/zipkin)

[服务链路追踪(Spring Cloud Sleuth)(Finchley版本)](https://blog.csdn.net/forezp/article/details/81041078)

[分布式链路跟踪 Sleuth 与 Zipkin【Finchley 版】](https://windmt.com/2018/04/24/spring-cloud-12-sleuth-zipkin/)
