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

### 限流

#### 限流组件说明

zuul网关项目通过集成[spring-cloud-zuul-ratelimit](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit)组件可以实现限流的功能，限流的方式有四种：ORIGIN(请求来源ip), USER(用户), URL(匹配url), ROLE(角色)，基本可以满足生产需求。并且不需要做任何代码修改，只需要添加限流组件的依赖及相关配置即可。

google官方的guava组件中的[RateLimiter](http://ifeve.com/guava-ratelimiter/)也可以实现限流。

#### spring-cloud-zuul-ratelimit限流示例

在zuul网关项目中添加限流组件，通过该组件+redis配合来完成限流控制

```xml
<!--熔断限流依赖包ratelimit组件+redis存储-->
<dependency>
    <groupId>com.marcosbarbero.cloud</groupId>
    <artifactId>spring-cloud-zuul-ratelimit</artifactId>
    <version>2.2.4.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

zuul网关代码不需要做任何修改，只需要添加限流组件及redis配置既可，本示例使用properties文件，yml文件示例参考限流组件[spring-cloud-zuul-ratelimit](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit)

```bash
# 【限流配置（示例按照ip或者url来限流）】
spring.redis.timeout=1000ms
spring.redis.database=13
spring.redis.host=172.17.7.189
spring.redis.port=6379
spring.redis.password=rQw@VzjA$106

zuul.ratelimit.enabled=true
# 指定redis存储的key前缀
zuul.ratelimit.key-prefix=ratelimit
zuul.ratelimit.behind-proxy=true
zuul.ratelimit.repository=REDIS

# 默认全局限流配置（这里指定60s同一ip请求次数不超过10次，并且请求总时间不超过10s，超过限制则收到httpcode=429错误）
#zuul.ratelimit.default-policy-list[0].limit=10
#zuul.ratelimit.default-policy-list[0].quota=10
#zuul.ratelimit.default-policy-list[0].refresh-interval=60
#zuul.ratelimit.default-policy-list[0].type=origin

# 指定某个服务的限流配置（注意：服务名称必须与zuul.routes.后面的名称一致）
zuul.ratelimit.policy-list.service-a[0].limit=10
zuul.ratelimit.policy-list.service-a[0].quota=10
zuul.ratelimit.policy-list.service-a[0].refresh-interval= 60
# 按照IP来限流
#zuul.ratelimit.policy-list.service-a[0].type=origin
# 按照url来限流（?后面的参数不作为key）--针对所有的url
#zuul.ratelimit.policy-list.service-a[0].type=url
# 按照url来限流（?后面的参数不作为key）--针对指定的的url（注意url不包含zuul的公共prefix以及服务名称，就是实际微服务controller中定义的url）
zuul.ratelimit.policy-list.service-a[0].type[0]=url=/hi

# 指定service-a服务的第二个url限流策略
#zuul.ratelimit.policy-list.service-a[1].limit=2
#zuul.ratelimit.policy-list.service-a[1].quota=2
#zuul.ratelimit.policy-list.service-a[1].refresh-interval= 10
#zuul.ratelimit.policy-list.service-a[1].type[0]=url=/hello
```

#### 自定义限流策略，去除限流配置文件配置

当限流策略非常多的时候，或者是需要将配置放到公司配置中心而不是本地配置文件的时候，可以使用java代码的方式来配置限流组件以及redis，同时也可以自定义限流策略（比如按照url或者header中的某个参数来限流），还可以在连接redis失败的时候自定义异常信息。

将application.properties中限流相关的配置及redis相关的配置全部注释掉，在zuul网关项目中添加一个专门的配置类

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.RateLimitKeyGenerator;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.RateLimitUtils;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.RateLimiter;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.properties.RateLimitProperties;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.properties.RateLimitRepository;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.properties.RateLimitType;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.repository.DefaultRateLimiterErrorHandler;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.repository.RateLimiterErrorHandler;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.config.repository.RedisRateLimiter;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.filters.RateLimitPreFilter;
import com.marcosbarbero.cloud.autoconfigure.zuul.ratelimit.support.DefaultRateLimitKeyGenerator;
import com.netflix.zuul.ZuulFilter;
import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
import org.springframework.cloud.netflix.zuul.filters.Route;
import org.springframework.cloud.netflix.zuul.filters.RouteLocator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.jedis.JedisClientConfiguration;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import org.springframework.web.util.UrlPathHelper;
import redis.clients.jedis.JedisPoolConfig;

import javax.servlet.http.HttpServletRequest;
import java.time.Duration;
import java.util.*;

@Configuration
public class RateLimitConfig {
    @Bean
    public ZuulFilter rateLimiterPreFilter(final RateLimiter rateLimiter, final RateLimitProperties rateLimitProperties,
                                           final RouteLocator routeLocator, final RateLimitKeyGenerator rateLimitKeyGenerator,
                                           final RateLimitUtils rateLimitUtils) {
        return new RateLimitPreFilter(rateLimitProperties, routeLocator, new UrlPathHelper(), rateLimiter,
                rateLimitKeyGenerator, rateLimitUtils);
    }

    @Bean
    public RedisRateLimiter redisRateLimiter(RateLimiterErrorHandler rateLimiterErrorHandler, RedisTemplate redisTemplate){
        return new RedisRateLimiter(rateLimiterErrorHandler,redisTemplate);
    }

    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        // 使用Jackson2JsonRedisSerialize 替换默认序列化
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);
        // 设置value的序列化规则和 key的序列化规则
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    @Bean
    public RedisConnectionFactory connectionFactory() {
        RedisProperties properties = new RedisProperties();
        properties.setHost("172.17.7.189");
        properties.setPort(6379);
        properties.setPassword("rQw@VzjA$106");
        properties.setDatabase(13);
        properties.setTimeout(Duration.ofMillis(60000));

        JedisPoolConfig poolConfig = new JedisPoolConfig();
        //最大活动对象数
        poolConfig.setMaxTotal(1000);
        //最大能够保持idel状态的对象数
        poolConfig.setMaxIdle(100);
        //最小能够保持idel状态的对象数
        poolConfig.setMinIdle(30);
        //当池内没有返回对象时，最大等待时间
        poolConfig.setMaxWaitMillis(10000);
        //调用borrow Object方法时，是否进行有效性检查
        poolConfig.setTestOnBorrow(true);
        //当调用return Object方法时，是否进行有效性检查
        poolConfig.setTestOnReturn(false);
        //向调用者输出“链接”对象时，是否检测它的空闲超时
        poolConfig.setTestWhileIdle(true);

        JedisClientConfiguration jedisClientConfiguration = null;
        if (properties.isSsl()){
            jedisClientConfiguration = JedisClientConfiguration.builder().usePooling().
                    poolConfig(poolConfig).and().
                    readTimeout(properties.getTimeout()).useSsl()
                    .build();
        }else {
            jedisClientConfiguration = JedisClientConfiguration.builder().usePooling().
                    poolConfig(poolConfig).and().
                    readTimeout(properties.getTimeout()).build();
        }

        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setDatabase(properties.getDatabase());
        redisStandaloneConfiguration.setPort(properties.getPort());
        redisStandaloneConfiguration.setPassword(RedisPassword.of(properties.getPassword()));
        redisStandaloneConfiguration.setHostName(properties.getHost());

        return new JedisConnectionFactory(redisStandaloneConfiguration, jedisClientConfiguration);
    }

    @Bean
    public RateLimitProperties rateLimitProperties(){
        RateLimitProperties properties = new RateLimitProperties();
        properties.setEnabled(true);
        properties.setKeyPrefix("ratelimit");
        properties.setBehindProxy(true);
        properties.setRepository(RateLimitRepository.REDIS);
        Map<String,List<RateLimitProperties.Policy>>  policyListMap = new HashMap<>();
        List<RateLimitProperties.Policy> policyList=new ArrayList<>();
        RateLimitProperties.Policy policy = new RateLimitProperties.Policy();
        policy.setLimit(10L);
        policy.setQuota(10L);
        policy.setRefreshInterval(60L);
        List<RateLimitProperties.Policy.MatchType> matchTypeList = new ArrayList<>();
        RateLimitProperties.Policy.MatchType matchType = new RateLimitProperties.Policy.MatchType(RateLimitType.URL,"/hi");
        matchTypeList.add(matchType);
        policy.setType(matchTypeList);
        policyList.add(policy);
        policyListMap.put("service-a",policyList);
        properties.setPolicyList(policyListMap);
        return properties;
    }

    @Bean
    public RateLimitUtils rateLimitUtils(){
        return new RateLimitUtils() {
            @Override
            public String getUser(HttpServletRequest request) {
                return null;
            }

            @Override
            public String getRemoteAddress(HttpServletRequest request) {
                return null;
            }

            @Override
            public Set<String> getUserRoles() {
                return null;
            }
        };
    }

    /**
     * 自定义限流的redis key，可以用来拼装请求中的url参数或者header，自定义限流策略
     * @param properties
     * @param rateLimitUtils
     * @return
     */
    @Bean
    public RateLimitKeyGenerator rateLimitKeyGenerator(RateLimitProperties properties, RateLimitUtils rateLimitUtils) {
        return new DefaultRateLimitKeyGenerator(properties, rateLimitUtils) {
            @Override
            public String key(HttpServletRequest request, Route route, RateLimitProperties.Policy policy) {

                String key = super.key(request, route, policy) + "_" + request.getMethod();
                System.out.println("custom ratelimit key: " + key);
                return key;
            }
        };
    }

    /**
     * 自定义错误处理逻辑，当存取redis出问题的时候可以用来记录一些日志之类的
     * @return
     */
    @Bean
    public RateLimiterErrorHandler rateLimitErrorHandler() {
        return new DefaultRateLimiterErrorHandler() {
            @Override
            public void handleSaveError(String key, Exception e) {
                System.out.println(String.format("DefaultRateLimiterErrorHandler-handleSaveError, key: %s, exception msg:%s",key,e.getMessage()));
            }

            @Override
            public void handleFetchError(String key, Exception e) {
                System.out.println(String.format("DefaultRateLimiterErrorHandler-handleFetchError, key: %s, exception msg:%s",key,e.getMessage()));
            }

            @Override
            public void handleError(String msg, Exception e) {
                System.out.println(String.format("DefaultRateLimiterErrorHandler-handleError, msg: %s, exception msg:%s",msg,e.getMessage()));
            }
        };
    }
}
```

因为去除了RedisTemplate相关的配置（spring.redis.xxxxxx），需要手动集成JedisConnectionFactory，所以pom文件中需要添加jedis依赖

```xml
<!--纯代码配置限流组件及redis存储时，需要依赖jedis包-->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.10.2</version>
</dependency>
```

#### 测试限流

1.启动eureka注册中心springcloud.f.eureka.server

2.启动测试服务springcloud.f.zuul.service-a，我们仅仅使用service-a这个服务来进行测试

3.启动zuul网关springcloud.f.zuul.gateway

4.访问以下zuul地址，通过zuul来访问service-a这个服务，不断的请求这个接口，由于我们配置的限流策略是60s内访问`/hi`超过10次就限流，所以当访问第11次的时候，zuul网关接口就返回了429错误

http://localhost:8961/api/api-a/hi?name=zhangsan&token=abc

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[路由网关(zuul)(Finchley版本)](https://blog.csdn.net/forezp/article/details/81041012)

[Zuul 超时、重试、并发参数设置](https://blog.csdn.net/xx326664162/article/details/83625104)

[spring-cloud服务网关中的Timeout设置](http://www.deanwangpro.com/2018/04/13/zuul-hytrix-ribbon-timeout/)

[spring-cloud-zuul-ratelimit](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit)

[Rate Limiting in Spring Cloud Netflix Zuul](https://www.baeldung.com/spring-cloud-zuul-rate-limit)
