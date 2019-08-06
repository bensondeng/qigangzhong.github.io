---
layout: post
title:  "springcloud-hystrix(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 介绍

断路器的功能可以有效防止由于某个服务超时或者停止服务导致依赖服务的连锁雪崩效应。

使用feign进行服务通信的时候，默认集成了hystrix断路器功能，但是默认是关闭的，需要在服务调用方配置中显示开启：

```bash
# 打开feign默认集成的hystrix断路器功能
feign.hystrix.enabled=true
```

如果使用ribbon+restTemplate的方式，需要手动添加pom依赖，入口类添加`@EnableHystrix`注解，需要断路器功能的方法上面添加`@HystrixCommand(fallbackMethod = "demoMethod")`注解。

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.hystrix)

## feign+hystrix进行客户端熔断

### 服务端

就是一个普通的服务，仅依赖springboot+eureka，服务接口我们模拟一下超时情况

```java
@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String sayHi(@RequestParam("name") String name) throws InterruptedException {

        //模拟超时
        Random random = new Random();
        int r = random.nextInt(5);
        System.out.println(r);
        TimeUnit.SECONDS.sleep(r);

        return String.format("Hi %s，i'm from port %s",name,port);
    }
}
```

### hystrix客户端

pom依赖与普通的feign客户端依赖一样，默认包含了hystrix功能

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
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

入口函数没什么需要特殊修改的

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class ServiceHiClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceHiClientApplication.class,args);
    }
}
```

定义的feign服务代理需要修改，指定熔断措施fallback

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
@FeignClient(value = "service-hi-hystrix", fallback = ServiceHiHystrix.class)
public interface ServiceHi {
    @GetMapping("/hi")
    String callHiService(@RequestParam("name") String name);
}


import org.springframework.stereotype.Component;
@Component
public class ServiceHiHystrix implements ServiceHi {
    @Override
    public String callHiService(String name) {
        return "sorry "+ name+", something is wrong, that's why you see this page";
    }
}
```

## hystrix-dashboard监控

上面的示例是在feign客户端使用hystrix断路器，hystrix-dashboard可以配置在服务端，通过dashboard可以观察到改服务的流量访问情况。hystrix-dashboard相当于是hystrix断路数据的消费方。

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.hystrix/springcloud.f.hystrix.dashboard.service-hi)

### 改造微服务服务端，添加hystrix服务端熔断支持以及hystrix-dashboard

微服务服务pom依赖中除去springboot，eureka依赖还需要加上hystrix-dashboard相关的3个依赖

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

    <!--hystrix-dashboard相关依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>
</dependencies>
```

入口类添加`@EnableHystrix`，`@EnableHystrixDashboard`注解

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;

@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrix
@EnableHystrixDashboard
public class HystrixDashboardServiceHiApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardServiceHiApplication.class,args);
    }
}
```

需要进行服务端熔断机制的接口上添加`@HystrixCommand(fallbackMethod = "xxx")`注解

```java
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    @HystrixCommand(fallbackMethod = "sayHiError")
    public String sayHi(@RequestParam("name") String name) throws InterruptedException {

        //模拟随机超时
        /*Random random = new Random();
        int r = random.nextInt(5);
        System.out.println(r);
        TimeUnit.SECONDS.sleep(r);*/

        return String.format("Hi %s，i'm from port %s",name,port);
    }

    public String sayHiError(String name){
        return String.format("Oops! %s, something goes wrong! I'm from port %s",name,port);
    }
}
```

同时，配置文件中需要暴露hystrix-dashboard需要使用到的url端点（默认是关闭的）

```bash
spring.application.name=service-hi-hystrix-dashboard
server.port=8861
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5

# 放开所有的web端点，包括/actuator/hystrix.stream, /actuator/health, /actuator/info
management.endpoints.web.exposure.include=*
management.endpoints.web.cors.allowed-origins=*
management.endpoints.web.cors.allowed-methods=*
```

### 测试hystrix-dashboard

1.启动eureka注册中心springcloud.f.eureka.server

2.启动微服务springcloud.f.hystrix.dashboard.service-hi

3.访问hystrix-dashboard的web界面，输入本服务的hystrix.stream接口地址

hystrix-dashboard的web界面：http://localhost:8861/hystrix

hystrix.stream接口地址：http://localhost:8861/actuator/hystrix.stream

访问添加`@HystrixCommand`注解的api接口，可以看到图形界面上的流量信息

http://localhost:8861/hi?name=zhangsan

## hystrix-dashboard turbine聚合监控

每个微服务都集成hystrix-dashboard之后，如何统一查看所有服务的监控数据呢？turbine就是对所有服务的hystrix-dashboard监控数据进行整合。

为了查看多个微服务，再创建一个独立的微服务并集成hystrix-dashboard，这样就有2个服务了

* springcloud.f.hystrix.dashboard.service-hi
* springcloud.f.hystrix.dashboard.service-hello

### 创建独立turbine站点，用来聚合所有微服务的hystrix-dashboard监控数据

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.hystrix/springcloud.f.hystrix.dashboard.service-turbine)

pom依赖如下

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

    <!--hystrix-dashboard相关依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>

    <!--turbine相关依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
    </dependency>
</dependencies>
```

入口类添加相关注解

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.cloud.netflix.turbine.EnableTurbine;

@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrix
@EnableHystrixDashboard
@EnableTurbine
public class TurbineApplication {
    public static void main(String[] args) {
        SpringApplication.run(TurbineApplication.class,args);
    }
}
```

再添加配置文件

```bash
spring.application.name=service-turbine
server.port=8866
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5

# 放开所有的web端点，包括/actuator/hystrix.stream, /actuator/health, /actuator/info
management.endpoints.web.exposure.include=*
management.endpoints.web.cors.allowed-origins=*
management.endpoints.web.cors.allowed-methods=*

# turbine相关配置
# 配置需要聚合的服务service-id列表
turbine.app-config=service-hi-hystrix-dashboard,service-hello-hystrix-dashboard
turbine.aggregator.cluster-config=default
turbine.cluster-name-expression=new String("default")
turbine.combine-host-port=true
turbine.instanceUrlSuffix.default=actuator/hystrix.stream
```

这个服务仅仅是用来聚合其它微服务的hystrix-dashboard监控数据，不需要其它业务逻辑代码

### 测试turbine

1.启动eureka注册中心springcloud.f.eureka.server

2.启动微服务springcloud.f.hystrix.dashboard.service-hi、springcloud.f.hystrix.dashboard.service-hello

3.启动turbine站点springcloud.f.hystrix.dashboard.service-turbine

4.开启turbine站点的hystrix-dashboard的web界面，添加turbine.stream接口的监控

hystrix-dashboard的web界面：http://localhost:8866/hystrix

turbine.stream接口地址：http://localhost:8866/actuator/turbine.stream

访问两个微服务的接口，可以看到turbine站点hystrix-dashboard图形界面上出现了两个微服务的监控信息

http://localhost:8861/hi?name=zhangsan

http://localhost:8862/hello?name=lisi

## hystrix监控数据持久化

hystrix+InfluxDB+grafana

参考：[采集Hystrix线程池指标并使用influxDB+Grafana实时监控（HystrixDashboard升级方案）](https://blog.csdn.net/wk52525/article/details/90550171)

## 类似hystrix的产品

[alibaba Sentinel](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D)，也可以支持springcloud

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[A-G各个版本的release notes](https://github.com/spring-projects/spring-cloud/wiki)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[Hystrix官方说明-Metrics and Monitoring](https://github.com/Netflix/Hystrix/wiki/Metrics-and-Monitoring)

[断路器（Hystrix）(Finchley版本)](https://blog.csdn.net/forezp/article/details/81040990)

[断路器监控(Hystrix Dashboard)(Finchley版本)](https://blog.csdn.net/forezp/article/details/81041113)

[Turbine(Finchley版本)](https://www.fangzhipeng.com/springcloud/2018/08/13/sc-f13-turbine.html)

[fallback & fallbackFactory](https://www.jianshu.com/p/3d804e99e613)
