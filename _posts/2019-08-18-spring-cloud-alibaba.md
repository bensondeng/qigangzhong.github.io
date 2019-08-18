---
layout: post
title:  "spring-cloud-alibaba"
categories: springcloud
tags:  微服务 springcloud spring-cloud-alibaba
author: 网络
---

* content
{:toc}

总结spring-cloud-alibaba的基础使用方法







## 版本说明

从[官方版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)可以看出来，目前孵化器版本支持SpringCloud Finchley版本的是0.2.2.RELEASE。根据[阿里的消息](https://yq.aliyun.com/articles/711938?utm_content=g_1000069851&do=login&accounttraceid=14ffb0b9-1e04-47dd-8b42-585716dc2403)，目前apache孵化版本还没有正式毕业，还不推荐生产使用，生产还是建议使用提交apache之前的版本。

## Nacos

### 安装启动

从[Nacos发布页面](https://github.com/alibaba/nacos/releases)下载Nacos服务器安装包，这里下载最新1.1.3版本

```bash
tar -zxvf nacos-server-1.1.3.tar.gz
cd nacos
# 单机模式启动
sh ./bin/startup.sh -m standalone
```

访问`http://192.168.255.138:8848/nacos/`，默认用户名密码nacos/nacos。

### nacos集群搭建

[Nacos三种部署模式](https://nacos.io/zh-cn/docs/deployment.html)

### nacos作为注册中心的代码示例

[代码示例](https://gitee.com/qigangzhong/springcloud.alibaba/tree/master/springcloud.alibaba.nacos)

#### 服务

pom引用

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

入口类添加@EnableDiscoveryClient注解，向注册中心注册服务

服务controller代码很简单

```java
@Slf4j
@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String hi(@RequestParam String name) {
        log.info("invoked name = " + name);
        return String.format("hi %s, i'm from port %s",name,port);
    }
}
```

配置文件中指定nacos注册中心地址

```bash
spring.application.name=service-hi
server.port=8001
spring.cloud.nacos.discovery.server-addr=192.168.255.138:8848
```

#### 客户端

客户端调用服务端有4种方式，都可以实现负载均衡

pom引用

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>

    <!--使用WebClient.Builder的依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <!--使用feign的依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

入口类添加@EnableDiscoveryClient来向nacos注册中心注册，如果使用feign客户端的话需要添加@EnableFeignClients注解扫描feign代理接口

```java
@EnableDiscoveryClient
@SpringBootApplication
@EnableFeignClients
public class ServiceHiClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceHiClientApplication.class,args);
    }
}
```

controller中使用4种方式来调用服务端

```java
@Slf4j
@RestController
public class HiController {
    @Autowired
    LoadBalancerClient loadBalancerClient;

    @GetMapping("/hi-1")
    public String hi1(){
        //1. 通过spring cloud负载均衡客户端来调用服务
        ServiceInstance serviceInstance = loadBalancerClient.choose("service-hi");
        String url = serviceInstance.getUri() + "/hi?name=john";
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject(url, String.class);
    }


    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    @Autowired
    RestTemplate restTemplate;

    @GetMapping("/hi-2")
    public String hi2(){
        //2. 通过负载均衡RestTemplate来调用服务
        return restTemplate.getForObject("http://service-hi/hi?name=john",String.class);
    }


    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
    @Autowired
    private WebClient.Builder webClientBuilder;

    @GetMapping("/hi-3")
    public Mono<String> hi3(){
        //3. 通过spring5提供的webclient来调用服务
        // 需要引入spring-boot-starter-webflux依赖包
        return webClientBuilder.build().get().uri("http://service-hi/hi?name=john").retrieve().bodyToMono(String.class);
    }


    @Autowired
    ServiceHi serviceHi;

    @GetMapping("/hi-4")
    public String hi4(){
        //4. 通过Feign客户端来调用服务
        // 应用入口类添加@EnableFeignClients开启Feign客户端扫描
        // 定义一个service-hi服务的feign代理接口
        // @Autowired注入接口实例
        return serviceHi.hi("john");
    }
}
```

使用第4种方式feign客户端时定义的feign客户端接口如下

```java
@FeignClient("service-hi")
public interface ServiceHi{
    @GetMapping("/hi")
    String hi(@RequestParam(name = "name") String name);
}
```

配置文件中同样指定nacos注册中心地址

```bash
spring.application.name=service-hi-client
server.port=8003
spring.cloud.nacos.discovery.server-addr=192.168.255.138:8848
```

#### 测试

1.启动nacos服务器，使用nacos/nacos登录管理控制台`http://192.168.255.138:8848/nacos/`

2.分别用2个端口8001,8002启动服务service-hi，观察nacos管理控制台服务列表

3.启动客户端service-hi-client，观察nacos管理控制台服务列表

4.调用service-hi-client接口测试4种方式调用服务

```bash
curl http://localhost:8003/hi-1
curl http://localhost:8003/hi-2
curl http://localhost:8003/hi-3
curl http://localhost:8003/hi-4
```

### nacos作为配置中心的代码示例

#### 配置中心添加配置

在nacos管理控制台配置列表菜单页面，点击右侧加号添加配置项，Data ID填写一个名称（xxx.properties），这个名称xxx需要跟即将使用这个配置的应用的名称保持一致，这个应用会使用这个唯一的配置项；Group保持默认，配置格式选择properties，这样一个应用就可以使用这个配置项了，我们在里面随便写几个配置信息：

```bash
testKey1=testValue1
testKey2=testValue2
```

配置中心的几个概念：

* Data ID

服务本地配置文件中可以指定使用哪个配置项，不指定的话会使用默认的配置项（就是${spring.application.name}.properties）

```bash
# 默认配置文件名称就是应用名称
spring.cloud.nacos.config.prefix=${spring.application.name}
# 默认配置文件扩展名为properties
spring.cloud.nacos.config.file-extension=properties
```

* Group

可以指定配置分组，不指定的话使用默认值`DEFAULT_GROUP`，这个分组可以用来解决重名的问题

```bash
spring.cloud.nacos.config.group=DEFAULT_GROUP
```

#### 应用中使用配置

pom依赖中引入nacos-config组件

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!--nacos注册中心客户端依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <!--nacos配置中心客户端依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

入口类很简单，仅启用服务发现

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ConfigClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class,args);
    }
}
```

controller中直接通过@Value来注入配置信息，为了能够实现配置的实时刷新，还需要添加@RefreshScope注解

```java
@RestController
@RefreshScope
public class ConfigController {
    @Value("${testKey1}")
    String testValue1;

    @GetMapping("/getTestKey1")
    public String getTestKey1(){
        return testValue1;
    }
}
```

配置文件中需要指定注册中心和配置中心地址，而且为了本地配置文件的加载顺序在最前，本地配置文件名称需要使用`bootstrap.properties`

```bash
spring.application.name=config-client
server.port=8004
spring.cloud.nacos.discovery.server-addr=192.168.255.138:8848
spring.cloud.nacos.config.server-addr=192.168.255.138:8848
```

#### 多环境配置

http://blog.didispace.com/spring-cloud-alibaba-nacos-config-2/

## 参考

[spring-cloud-alibaba-wiki](https://github.com/alibaba/spring-cloud-alibaba/wiki)

[spring-cloud-alibaba-examples](https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples)

[使用Nacos实现服务注册与发现](http://blog.didispace.com/spring-cloud-alibaba-1/)

[nacos文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)
