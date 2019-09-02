---
layout: post
title:  "spring-cloud-alibaba-gateway"
categories: springcloud
tags:  微服务 springcloud spring-cloud-alibaba gateway
author: 网络
---

* content
{:toc}

总结spring-cloud-alibaba使用gateway的方法







## 介绍

## 路由

### 创建测试微服务项目

[测试微服务项目](https://gitee.com/qigangzhong/springcloud.alibaba/tree/master/springcloud.alibaba.gateway/springcloud.alibaba.gateway.service-hi)就是一个普通的微服务项目，注册到nacos注册中心。

pom添加nacos服务发现依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--服务发现依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--lombok依赖-->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

入口类没什么特别的，接口也很简单

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SCGServiceHiApplication {
    public static void main(String[] args) {
        SpringApplication.run(SCGServiceHiApplication.class,args);
    }
}


@Slf4j
@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String sayHi(@RequestParam("name") String name) {
        return String.format("Hi %s, i'm from service-a, port %s",name,port);
    }
}
```

配置文件只要指定nacos注册中心地址就可以了

```bash
spring.application.name=service-hi
server.port=8861
# nacos注册中心地址
spring.cloud.nacos.discovery.server-address=192.168.237.128:8848
```

### 创建测试网关项目

[测试网关项目](https://gitee.com/qigangzhong/springcloud.alibaba/tree/master/springcloud.alibaba.gateway/springcloud.alibaba.gateway.server)是一个spring-cloud-gateway项目，注册到nacos注册中心，通过配置文件来配置路由规则，当然也可以通过代码的方式配置，参考[spring-cloud-gateway示例](https://qigangzhong.github.io/2019/08/20/springcloud-gateway/)。

pom依赖中依赖gateway starter，nacos注册中心即可，启用actuator可以通过`/actuator/gateway/routes`端点来查看所有的路由信息

```xml
<dependencies>
    <!--spring-cloud-gateway依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!--服务发现依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--actuator依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!--lombok依赖-->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

入口类和普通微服务一样

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SCGServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SCGServerApplication.class,args);
    }
}
```

配置文件

```bash
spring.application.name=scg-server
server.port=8900
# nacos注册中心地址
spring.cloud.nacos.discovery.server-addr=192.168.237.128:8848

# 打印debug日志，方便调试跟踪问题
logging.level.org.springframework.cloud.gateway=debug

# 启用基于服务发现的路由定位
spring.cloud.gateway.discovery.locator.enabled=true
# 启用服务实例id名称小写支持
spring.cloud.gateway.discovery.locator.lowerCaseServiceId=true

# 启用gateway这个端点
management.endpoint.gateway.enabled=true
#management.endpoints.web.exposure.include=gateway
management.endpoints.web.exposure.include=*

# 路由规则配置
spring.cloud.gateway.routes[0].id=service_hi_route
spring.cloud.gateway.routes[0].uri=lb://service-hi
spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1
spring.cloud.gateway.routes[0].predicates[0]=Path=/service-hi/**
```

具体的路由规则配置方式与普通的spring-cloud-gateway无异，参考[spring-cloud-gateway示例](https://qigangzhong.github.io/2019/08/20/springcloud-gateway/)。

### 测试

1.启动nacos注册中心

2.启动微服务springcloud.alibaba.gateway.service-hi

3.启动网关项目springcloud.alibaba.gateway.server

4.通过网关来访问微服务`http://localhost:8900/service-hi/hi?name=abc`

## 过滤器

### 全局过滤器

在上面的网关测试项目springcloud.alibaba.gateway.server中添加一个全局鉴权过滤器，访问url参数必须带token才能通过

```java
/**
 * 全局鉴权过滤器
 */
@Component
public class AuthFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getQueryParams().getFirst("token");

        if (token == null || token.isEmpty()) {
            ServerHttpResponse response = exchange.getResponse();

            // 封装错误信息
            Map<String, Object> responseData = Maps.newHashMap();
            responseData.put("code", 401);
            responseData.put("message", "非法请求");
            responseData.put("cause", "Token is empty");

            try {
                // 将信息转换为 JSON
                ObjectMapper objectMapper = new ObjectMapper();
                byte[] data = objectMapper.writeValueAsBytes(responseData);

                // 输出错误信息到页面
                DataBuffer buffer = response.bufferFactory().wrap(data);
                response.setStatusCode(HttpStatus.UNAUTHORIZED);
                response.getHeaders().add("Content-Type", "application/json;charset=UTF-8");
                return response.writeWith(Mono.just(buffer));
            } catch (JsonProcessingException e) {
                e.printStackTrace();
            }
        }

        return chain.filter(exchange);
    }

    /**
     * 设置过滤器的执行顺序
     * @return
     */
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

重启网关项目springcloud.alibaba.gateway.server，访问`http://localhost:8900/service-hi/hi?name=abc`则报非法请求错误，添加`&token=xxx`则可以正常访问。

### Gateway过滤器

## 限流

### 集成sentinel来进行限流

sentinel提供了对SCG的适配组件，

pom文件添加sentinel相关依赖

```xml
<!--sentinel依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<!--sentinel针对gateway的适配器-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-cloud-gateway-adapter</artifactId>
    <version>1.6.0</version>
</dependency>
<!--sentinel组件依赖servlet-api-->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
</dependency>
```

添加一个配置java类

```java
import com.alibaba.csp.sentinel.adapter.gateway.sc.SentinelGatewayFilter;
import com.alibaba.csp.sentinel.adapter.gateway.sc.exception.SentinelGatewayBlockExceptionHandler;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.http.codec.ServerCodecConfigurer;
import org.springframework.web.reactive.result.view.ViewResolver;
import java.util.Collections;
import java.util.List;

@Configuration
public class GatewayConfiguration {

    private final List<ViewResolver> viewResolvers;
    private final ServerCodecConfigurer serverCodecConfigurer;

    public GatewayConfiguration(ObjectProvider<List<ViewResolver>> viewResolversProvider,
                                ServerCodecConfigurer serverCodecConfigurer) {
        this.viewResolvers = viewResolversProvider.getIfAvailable(Collections::emptyList);
        this.serverCodecConfigurer = serverCodecConfigurer;
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public SentinelGatewayBlockExceptionHandler sentinelGatewayBlockExceptionHandler() {
        // Register the block exception handler for Spring Cloud Gateway.
        return new SentinelGatewayBlockExceptionHandler(viewResolvers, serverCodecConfigurer);
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public GlobalFilter sentinelGatewayFilter() {
        return new SentinelGatewayFilter();
    }
}
```

配置文件添加sentinel地址配置即可

```bash
# sentinel通信地址以及dashboard地址
spring.cloud.sentinel.transport.port=8719
spring.cloud.sentinel.transport.dashboard=127.0.0.1:9001
```

### 测试

1.开启nacos，sentinel

2.开启网关项目springcloud.alibaba.gateway.server，开启微服务项目springcloud.alibaba.gateway.service-hi

3.访问`http://localhost:8900/service-hi/hi?name=test&token=abc`，然后观察sentinel-dashboard上的实时监控

4.sentinel-dashboard上簇点链路菜单下可以看到路由信息，添加流控和降级规则后再次访问url观察效果

## 熔断降级

## 参考

[sentinel网关限流](https://github.com/alibaba/Sentinel/wiki/%E7%BD%91%E5%85%B3%E9%99%90%E6%B5%81)
