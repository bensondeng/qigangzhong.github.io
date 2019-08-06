---
layout: post
title:  "springboot actuator & admin"
categories: springcloud
tags:  微服务 springcloud springboot
author: 网络
---

* content
{:toc}









## spring-boot-actuator

spring-boot-actuator是一个用来监控和收集springboot应用的运行情况的工具，我们可以利用它提供的restApi的[endpoint端点](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)来了解应用程序的信息，也可以自定义端点。

spring boot2.x中，默认只开放了info、health两个端点，其余的需要自己通过配置management.endpoints.web.exposure.include属性来加载（有include自然就有exclude）。如果想单独操作某个端点可以使用management.endpoint.端点.enabled属性进行启用或者禁用。

springboot应用集成actuator的方法很简单，pom中引用依赖包：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

配置中放开端点：

```bash
# 放开所有的web端点，包括/actuator/hystrix.stream, /actuator/health, /actuator/info等
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=ALWAYS
management.endpoints.web.cors.allowed-origins=*
management.endpoints.web.cors.allowed-methods=*
```

## spring-boot-admin

[spring-boot-admin](https://github.com/codecentric/spring-boot-admin)在actuator的基础上提供了一个可视化的web ui。

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/sba)

### admin-server

下面是基于springcloud环境搭建一个sba-server端站点。

首先添加相关pom依赖：

```xml
 <dependencies>
    <!--eureka及springboot web依赖包-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--admin主依赖包-->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
        <version>2.0.3</version>
    </dependency>

    <!--接入actuator，用来监控自身-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!--admin提供登陆功能依赖包-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

入口类添加`@EnableAdminServer`注解：

```java
import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableAdminServer
public class SbaAdminApplication {
    public static void main(String[] args) {
        SpringApplication.run(SbaAdminApplication.class,args);
    }
}
```

为了提供登陆功能，保护一些敏感信息，需要一些spring security的配置：

```java
import de.codecentric.boot.admin.server.config.AdminServerProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;

@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {

    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminContextPath + "/");

        http.authorizeRequests()
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .antMatchers(adminContextPath+"/actuator/**").permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                .logout().logoutUrl(adminContextPath + "/logout").and()
                .httpBasic().and()
                .csrf().disable();
        // @formatter:on
    }
}
```

配套的配置信息如下：

```bash
spring.application.name=sba-admin-server
server.port=18080
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/

# 登陆功能配置
spring.security.user.name=admin
spring.security.user.password=admin

# 放开所有的web端点，包括/actuator/hystrix.stream, /actuator/health, /actuator/info，可以让sba监控自身
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=ALWAYS
management.endpoints.web.cors.allowed-origins=*
management.endpoints.web.cors.allowed-methods=*
```

hystrix dashboard turbine在springcloud集群中可以很方便的监控微服务的流量信息，这个功能可以集成到sba上统一查看，但是hystrix dashboard功能单一生产环境价值不大，不作演示。

### 客户端

客户端当然要先接入actuator组件，按照文章开头的第一节中的方式引入pom依赖及配置信息。

对于springcloud中的服务，不需要额外引用sba-client组件，sba-server端注册到eureka注册中心之后会自动从注册中心拉取服务信息。

## 参考

[actuator服务监控与管理](https://www.cnblogs.com/baidawei/p/9183531.html)

[Spring Boot Admin 2.0](https://windmt.com/2018/05/22/spring-boot-admin-guide/)
