---
layout: post
title:  "springcloud-ribbon-feign(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 介绍

springcloud有两种服务调用方式：

* ribbon + restTemplate
* feign（更好的选择）

ribbon实现了客户端负载均衡，通过restTemplate调用服务端

feign采用基于接口的注解，默认集成了ribbon，所以也可以实现负载，并且整合了Hystrix，具有熔断的能力

## ribbon + restTemplate

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.ribbon)，注册中心使用[eureka示例](https://qigangzhong.github.io/2019/07/24/springcloud-eureka/)中的[单节点注册中心](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.eureka/springcloud.f.eureka.server)。



### 示例服务端

服务端没有特别的地方，就是一个普通的springboot restapi服务，注册到eureka注册中心，跟ribbon还没有任何关系

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
</dependencies>
```

```java
//入口类
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
@SpringBootApplication
@EnableDiscoveryClient
public class ServiceHiApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceHiApplication.class,args);
    }
}

//示例controller
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String sayHi(@RequestParam("name") String name){
        return String.format("Hi %s，i'm from port %s",name,port);
    }
}
```

```bash
spring.application.name=service-hi-ribbon
server.port=8861
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

首先启动[注册中心](https://qigangzhong.github.io/2019/07/24/springcloud-eureka/)，再启动两个示例服务端，端口分别为8861、8862。

### 示例ribbon客户端

客户端才集成ribbon组件，配合RestTemplate来调用服务端

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
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    </dependency>
</dependencies>
```

添加RestTemplate，使用@LoadBalanced来支持负载均衡

```java
//入口类
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;
@SpringBootApplication
@EnableDiscoveryClient
public class ServiceHiClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceHiClientApplication.class,args);
    }

    @Bean
    @LoadBalanced
    RestTemplate restTemplate(){
        return new RestTemplate();
    }
}

//示例controller，通过RestTemplate来调用服务端
//注意URL的格式
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;
@RestController
public class HiClientController {
    @Autowired
    RestTemplate restTemplate;

    @GetMapping("/callHiService")
    public String callHiService(){
        return restTemplate.getForObject("http://SERVICE-HI/hi?name=zhangsan",String.class);
    }
}
```

```bash
spring.application.name=service-hi-ribbon-client
server.port=8863
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

启动客户端，访问`http://localhost:8863/callHiService`，会看到返回结果在下面两个端口的服务之间轮询

```bash
# Hi zhangsan，i'm from port 8861
# Hi zhangsan，i'm from port 8862
```

## feign

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.feign)，注册中心使用[eureka示例](https://qigangzhong.github.io/2019/07/24/springcloud-eureka/)中的[单节点注册中心](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.eureka/springcloud.f.eureka.server)。

### 示例服务端

同上一节示例服务端，没有特别的地方，就是一个普通的springboot restapi服务，注册到eureka注册中心，跟feign还没有任何关系

同样首先启动[注册中心](https://qigangzhong.github.io/2019/07/24/springcloud-eureka/)，再启动两个示例服务端，端口分别为8861、8862。

### 示例feign客户端

maven添加feign依赖

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

```java
//入口类添加@EnableFeignClients注解
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

//添加一个服务端的feign代理接口，指定服务端的应用名称，接口地址，参数
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
@FeignClient(value = "service-hi-feign")
public interface ServiceHi {
    @GetMapping("/hi")
    String callHiService(@RequestParam("name") String name);
}

//在controller中调用这个feign代理接口
import com.qigang.springcloud.f.feign.servicehi.client.proxy.ServiceHi;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class HiClientController {
    @Autowired
    ServiceHi serviceHi;

    @GetMapping("/callHiService")
    public String callHiService(){
        return serviceHi.callHiService("zhangsan");
    }
}
```

```bash
spring.application.name=service-hi-feign-client
server.port=8863
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

启动客户端，访问`http://localhost:8863/callHiService`，会看到返回结果在下面两个端口的服务之间轮询，结果与ribbin+restTemplate的方式没有区别

```bash
# Hi zhangsan，i'm from port 8861
# Hi zhangsan，i'm from port 8862
```

### 修改负载均衡策略、超时时间

配置的方式：

```bash
# 以配置的方式来定义ribbon的负载均衡策略，超时时间等，这个是为某个服务单独配置的
# 修改服务的负载均衡策略
service-hi-feign.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.BestAvailableRule
# 请求连接超时时间
service-hi-feign.ribbon.ConnectTimeout=3000
# 请求处理超时时间
service-hi-feign.ribbon.ReadTimeout=3000
# 对所有请求都进行重试，false代表只对GET请求重试，对于写数据接口必须做幂等才行，谨慎配置！
service-hi-feign.ribbon.OkToRetryOnAllOperations=true
# 切换实例的重试次数，不包括首次调用
service-hi-feign.ribbon.MaxAutoRetriesNextServer=2
# 对当前实例的重试次数，不包括首次调用
service-hi-feign.ribbon.MaxAutoRetries=2
```

代码配置的方式如下，不过好像重试的配置没有办法区分每个实例的重试次数还有切换实例的重试次数：

```java
import com.netflix.loadbalancer.BestAvailableRule;
import com.netflix.loadbalancer.IRule;
import feign.Request;
import feign.Retryer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CustomRibbonConfig {
    /**
     * 负载均衡策略
     */
    @Bean
    public IRule ribbonRule(){
        return new BestAvailableRule();
    }

    //region 超时、重试次数设置
    public static int connectTimeOutMillis = 3000;
    public static int readTimeOutMillis = 3000;

    @Bean
    public Request.Options options() {
        return new Request.Options(connectTimeOutMillis, readTimeOutMillis);
    }

    @Bean
    public Retryer feignRetryer(){
        Retryer retryer = new Retryer.Default(100, 1000, 2);
        return retryer;
    }
    //endregion
}
```

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[失败请求重试说明](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html#retrying-failed-requests)

[Spring Cloud各组件重试总结](http://www.itmuch.com/spring-cloud-sum/spring-cloud-retry/)

[服务消费者（rest+ribbon）(Finchley版本)](https://blog.csdn.net/forezp/article/details/81040946)

[服务消费者（Feign）(Finchley版本)](https://blog.csdn.net/forezp/article/details/81040965)
