---
layout: post
title:  "spring-cloud-alibaba-sentinel"
categories: springcloud
tags:  微服务 springcloud spring-cloud-alibaba sentinel
author: 网络
---

* content
{:toc}

总结spring-cloud-alibaba的基础使用方法







## 下载安装sentinel-dashboard

[下载地址](https://github.com/alibaba/Sentinel/releases) | [文档地址](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D)

下载1.6.0的jar包，在centos7服务器上启动

```bash
java -Dserver.port=9001 -jar sentinel-dashboard-1.6.0.jar

# 后台启动
nohup java -Dserver.port=9001 -jar sentinel-dashboard-1.6.0.jar >> app.log 2>&1 &

# 常用java -jar启动参数
-Dsentinel.dashboard.auth.username=sentinel
-Dsentinel.dashboard.auth.password=sentinel
-Dserver.servlet.session.timeout=7200  # 单位s
```

打开`http://192.168.237.128:9001`使用sentinel/sentinel登陆

## 服务接入sentinel

### 接入

pom依赖

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

    <!--sentinel依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

入口类就是普通的springboot入口类，添加@EnableDiscoveryClient服务发现注解

```java
@EnableDiscoveryClient
@SpringBootApplication
public class SentinelRateLimitApplication {
    public static void main(String[] args) {
        SpringApplication.run(SentinelRateLimitApplication.class,args);
    }
}
```

controller就是普通的controller，没有任何特殊的地方

```java
@RestController
public class HiController {
    @GetMapping("/hi")
    public String hi(){
        return "hi world";
    }
}
```

配置文件中需要指定sentinel的地址

```bash
spring.application.name=sentinel-rate-limiting
server.port=8005
# sentinel地址
spring.cloud.sentinel.transport.dashboard=192.168.237.128:9001
# nacos注册中心地址
spring.cloud.nacos.discovery.server-addr=192.168.237.128:8848
```

### 测试

访问`http://192.168.237.128:8005/hi`，观察sentinel-dashboar实时流量监控`http://192.168.237.128:9001/#/dashboard/metric/sentinel-rate-limiting`

## 接口级别限流

在sentinel控制台左侧菜单选择“簇点链路”菜单，可以看到访问的接口路径`/hi`，右侧有“流控”、“降级”、“热点”、“授权”4个按钮，点击“流控”可以针对接口设置QPS限制。

默认情况下，设置的限流策略存储在内存，如果重启sentinel服务器，这些规则就消失了。sentinel有4中存储规则的方式：

* 文件
* zookeeper
* apollo
* nacos

### nacos存储限流规则

#### 改造springcloud应用

pom添加nacos存储依赖

```xml
<!--sentinel使用nacos存储规则时需要的nacos依赖-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <version>1.5.2</version>
</dependency>
```

本地配置文件添加配置

```bash
# sentinel使用nacos存储规则信息
spring.cloud.sentinel.datasource.ds.nacos.server-addr=192.168.237.128:8848
spring.cloud.sentinel.datasource.ds.nacos.dataId=${spring.application.name}-sentinel
spring.cloud.sentinel.datasource.ds.nacos.groupId=DEFAULT_GROUP
spring.cloud.sentinel.datasource.ds.nacos.rule-type=flow
```

#### 在nacos配置中心添加配置

上一步配置文件中指定了使用`dataId=${spring.application.name}-sentinel`的配置来存储限流规则，我们打开nacos配置中心`http://192.168.237.128:8848/nacos`，使用nacos/nacos登陆，在配置中心默认public命名空间下面，添加配置，Data ID=`sentinel-rate-limiting-sentinel`不带后缀，Group默认，配置格式Json，配置内容如下：

```json
[
    {
        "resource": "/hi",
        "limitApp": "default",
        "grade": 1,
        "count": 2,
        "strategy": 0,
        "controlBehavior": 0,
        "clusterMode": false
    }
]
```

> 配置项说明：
>
> * resource：资源名，即限流规则的作用对象
> * limitApp：流控针对的调用来源，若为 default 则不区分调用来源
> * grade：限流阈值类型（QPS 或并发线程数）；0代表根据并发数量来限流，1代表根据QPS来进行流量控制
> * count：限流阈值
> * strategy：调用关系限流策略
> * controlBehavior：流量控制效果（直接拒绝、Warm Up、匀速排队）
> * clusterMode：是否为集群模式

这个配置就跟上一个章节直接在sentinel控制台“簇点链路”手动添加“流控”这种方式是一样的效果，都是限制`/hi`接口的QPS=2。这样nacos中修改了配置也会即时同步到sentinel，即使服务重启也不会丢失。但是反过来如果修改sentinel中的规则不会同步回nacos配置中心，如果想实现sentinel控制台修改限流策略也同步到nacos配置中心的话，需要对sentinel-dashboard服务器端源码做扩展，参考[翟永超这篇教程](http://blog.didispace.com/spring-cloud-alibaba-sentinel-2-4/)。

#### 测试

重新启动应用，观察一下输出日志，确定nacos中配置的限流规则是否被加载

```bash
o.s.c.a.s.c.SentinelDataSourceHandler    : [Sentinel Starter] DataSource ds-sentinel-nacos-datasource load 1 FlowRule
```

访问`http://192.168.237.128:8005/hi`，观察sentinel控制台`http://192.168.237.128:9001`流控规则，会发现nacos中的流控配置已经同步到sentinel

## 方法级别限流、熔断降级

### 应用中使用AOP

方法级别的限流和熔断降级需要使用到SentinelResourceAspect这个切面，所以先注入这个切面

```java
/**
 * 用于方法级别的流控、熔断降级
 * @return
 */
@Bean
public SentinelResourceAspect sentinelResourceAspect() {
    return new SentinelResourceAspect();
}
```

添加一个内部service类，用来模拟方法级别的限流和降级，并创建controller来调用这个service，配置什么的不用动。

```java
@Slf4j
@Service
public class HelloService {

    //模拟限流处理
    @SentinelResource(value = "helloServiceMethod1", blockHandler = "hello1BlockHandler")
    public String hello1(String name){
        return "hello"+name;
    }
    //被限流之后的动作
    public String hello1BlockHandler(String name, BlockException blockException){
        log.error("hello1BlockedException, name="+name,blockException);
        return "oops, blockException!!!";
    }



    //模拟发生异常后降级处理
    @SentinelResource(value = "helloServiceMethod2", fallback = "hello2Fallback")
    public String hello2(String name){
        throw new RuntimeException("发生异常");
    }
    //被降级之后的动作
    public String hello2Fallback(String name){
        log.error("hello2Fallback, name="+name);
        return "oops, fallback!!!";
    }
}




/**
 * 测试方法级别的限流和熔断降级
 */
@Slf4j
@RestController
public class HelloController {
    @Autowired
    HelloService helloService;

    @GetMapping("/hello1")
    public String hello1(){
        return helloService.hello1("zhangsan "+ new Date());
    }

    @GetMapping("/hello2")
    public String hello2(){
        return helloService.hello2("lisi "+ new Date());
    }
}
```

### 测试

重启应用，访问`http://192.168.237.128:8005/hi`，观察sentinel控制台`http://192.168.237.128:9001`簇点链路，会发现多了刚才HelloService中的方法，可以在上面添加限流和降级的规则。同样的这个规则默认是保存在应用内存的，如果应用重启规则就消失了。需要修改sentinal-dashboard源码支持规则同步到nacos配置中心，参考[翟永超这篇教程](http://blog.didispace.com/spring-cloud-alibaba-sentinel-2-4/)。

## 参考

[spring-cloud-alibaba-wiki](https://github.com/alibaba/spring-cloud-alibaba/wiki)

[spring-cloud-alibaba-examples](https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples)

[Spring-Cloud-Alibaba 系列](http://blog.didispace.com/tags/Spring-Cloud-Alibaba/)
