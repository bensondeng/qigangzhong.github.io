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

## feign+hystrix

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

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[断路器（Hystrix）(Finchley版本)](https://blog.csdn.net/forezp/article/details/81040990)
