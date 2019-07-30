---
layout: post
title:  "springcloud-zuul(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 介绍

zuul的主要功能是路由转发和过滤器。可以利用zuul来做以下事情：

* 路由转发
* 负载均衡
* 请求鉴权
* 数据、性能监控

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.zuul)

## zuul网关搭建

### pom依赖

就是zuul配置，另外加一个spring-retry配置，用来做服务重试

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
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>
</dependencies>
```

### 入口类

就是添加一个`@EnableZuulProxy`注解

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
public class ZuulGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulGatewayApplication.class,args);
    }
}
```

### 自定义filter

自定义filter通过`shouldFilter`方法返回值即可控制启用或者禁用，如果是第三方的filter，可以通过配置来启用或者禁用

```bash
zuul.filterName.pre.disable=true
```

示例检测请求是否传递token参数的过滤器

```java
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import org.springframework.stereotype.Component;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

@Component
public class MyFilter extends ZuulFilter {
    /**
     * 过滤一共有4种类型：
     * 1. pre
     * 2. routing
     * 3. post
     * 4. error
     * @return
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 过滤器顺序
     * @return
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 判断是否要过滤，true代表永远过滤
     * @return
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();
        System.out.println("请求路径："+request.getRequestURL().toString());
        String token  = request.getParameter("token");
        if(token==null || token.length()<=0){
            System.out.println("没有传递token参数");
            context.setSendZuulResponse(false);
            context.setResponseStatusCode(401);
            try {
                context.getResponse().getWriter().write("token is empty");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        System.out.println("token参数为："+token);
        return null;
    }
}
```

### 路由熔断机制

可以通过自定义FallbackProvider来针对指定的服务做熔断处理，如果服务超时或者不可用，那记录异常信息，返回固定结果给服务调用方

```java
import org.springframework.cloud.netflix.zuul.filters.route.FallbackProvider;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.stereotype.Component;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

@Component
public class MyFallback implements FallbackProvider {
    /**
     * 指定需要对哪些服务作熔断处理
     * @return
     */
    @Override
    public String getRoute() {
        return "service-a";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        if (cause != null) {
            String causeMsg = cause.getMessage();
            System.out.println("fallback cause:"+causeMsg);
        }

        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "OK";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("The service is unavailable.".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}
```

### 重试

zuul集成了hystrix和ribbon，对于hystrix和ribbon都有各自的超时时间设置，建议是hystrix的超时时间大于ribbon的超时时间，因为由于hystrix超时触发熔断之后，可能会影响到ribbon的超时重试功能，实际测试其实不影响，但是如果设置hystrix超时时间小于ribbon的超时时间，运行时会出现一个warning

```log
2019-07-30 15:48:40.962  WARN 10016 --- [nio-8961-exec-6] o.s.c.n.z.f.r.s.AbstractRibbonCommand    : The Hystrix timeout of 100ms for the command service-a is set lower than the combination of the Ribbon read and connect timeout, 6000ms.
```

重试机制完全通过配置来实现，在zuul网关应用中添加超时相关配置实现，完整的配置文件如下：

```bash
spring.application.name=zuul-gateway
server.port=8961
# eureka地址配置，如果eureka是个集群，那么逗号隔开配置所有地址
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5

# 【路由配置】（通过service-id的方式来路由，不建议通过url的方式来路由）
# 以/api-a/ 开头的请求都转发给service-a服务，以/api-b/开头的请求都转发给service-b服务
zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.service-id=service-a
zuul.routes.api-b.path=/api-b/**
zuul.routes.api-b.service-id=service-b
# 忽略所有包含某关键词的请求
zuul.ignored-patterns=/**/abc/*
# 所有的请求必须带指定的前缀
zuul.prefix=/api
# 去掉前缀后的请求才转发给后端，默认就是true不用指定
zuul.strip-prefix=true

# 【重试配置】
zuul.retryable=true
# hystrix熔断超时时间需要大于ribbon超时时间，因为如果hystrix先于ribbon超时熔断了，ribbon的重试就没有意义了
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=100
# ribbon读超时时间
service-a.ribbon.ReadTimeout=1000
# ribbon连接超时时间
service-a.ribbon.ConnectTimeout=1000
# 对当前服务的重试次数，不包含当前请求
service-a.ribbon.MaxAutoRetries=2
# 切换相同Server的次数，不包含当前Server
service-a.ribbon.MaxAutoRetriesNextServer=0
# 默认false，代表只对GET请求重试，true代表对所有请求都重试（慎用！）
service-a.ribbon.OkToRetryOnAllOperations=false
# 也可以对指定路由进行重试配置（优先级比全局更高）
#zuul.routes.<routename>.retryable=true
# 根据HTTP响应码来重试
service-a.ribbon.retryableStatusCodes=404,502

# 当zuul路由直接使用url而不是service-id，那么ribbon的超时配置不生效，下面的zuul超时配置生效，参考：https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html#_zuul_timeouts
# zuul.host.socket-timeout-millis=1000
# zuul.host.connect-timeout-millis=1000
```

#### 测试重试

1.启动eureka注册中心springcloud.f.eureka.server

2.启动测试服务springcloud.f.zuul.service-a，我们仅仅使用service-a这个服务来进行测试

3.启动zuul网关springcloud.f.zuul.gateway

4.访问以下zuul地址，通过zuul来访问service-a这个服务，服务里面模拟超时，zuul网关发现访问服务超时，则执行自定义fallback操作返回一个指定的fallback消息，如果观察一下service-a服务日志，可以看到收到了重试的请求

http://localhost:8961/api/api-a/hi?name=zhangsan&token=abc

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[路由网关(zuul)(Finchley版本)](https://blog.csdn.net/forezp/article/details/81041012)

[Zuul 超时、重试、并发参数设置](https://blog.csdn.net/xx326664162/article/details/83625104)

[spring-cloud服务网关中的Timeout设置](http://www.deanwangpro.com/2018/04/13/zuul-hytrix-ribbon-timeout/)
