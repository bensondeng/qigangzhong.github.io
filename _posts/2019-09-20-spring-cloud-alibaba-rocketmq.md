---
layout: post
title:  "spring-cloud-alibaba-rocketmq"
categories: springcloud
tags:  微服务 springcloud spring-cloud-alibaba rocketmq
author: 网络
---

* content
{:toc}

总结spring-cloud-alibaba的基础使用方法







## 介绍

### 消息推拉模式

consumer与broker之间的消息通信一般有推（push）和拉（pull）两种方式。

* push：broker将消息推送给consumer
* pull：consumer主动向broker拉消息

### 在springcloud中使用rocketmq的三种方式

* 使用rocketmq原生api
* 使用springboot集成的方式
* 使用spring-cloud-stream的方式

使用这种方式来将rocketmq集成到springcloud项目中需要注意的是，spring-cloud-alibaba有两个分支版本：

![spring-cloud-alibaba-dependencies-version.jpg](/images/springcloud/spring-cloud-alibaba-dependencies-version.jpg)

示例项目中使用的是`org.springframework.cloud`这个分支，这个分支主要的两个版本为`0.2.2.RELEASE`是针对Finchley版本的spring-cloud，`0.9.0.RELEASE`是针对Greenwitch版本的spring-cloud。这两个版本都不支持pull消息拉取模式。

> 如果spring-cloud-alibaba项目使用的是`org.springframework.cloud`这个分支，代码中使用了pull模式，通过下面的方式来定义一个input管道，启动时会报错：
>
> ```java
> @Input("input5")
> PollableMessageSource input5();
> ```
>
> 原因是这两个版本中`spring-cloud-stream-binder-rocketmq`模块的`RocketMQMessageChannelBinder.java`类没都有实现父类`AbstractMessageChannelBinder`中的`createPolledConsumerResources`方法，默认情况下如果没有实现它，项目会直接抛出异常：
>
> ```java
> protected PolledConsumerResources createPolledConsumerResources(String name, String group,
>         ConsumerDestination destination, C consumerProperties) {
>     throw new UnsupportedOperationException("This binder does not support pollable consumers");
> }
> ```
>
> `com.alibaba.cloud`分支的`v2.0.0.RELEASE`版本中的`RocketMQMessageChannelBinder.java`才支持pull模式。
>
> 参考[官方github源码](https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-stream-binder-rocketmq/src/main/java/com/alibaba/cloud/stream/binder/rocketmq/RocketMQMessageChannelBinder.java)

## 通过spring-cloud-stream方式集成rocketmq

[示例项目](https://gitee.com/qigangzhong/springcloud.alibaba/tree/master/springcloud.alibaba.rocketmq)基于apache版本的spring-cloud-alibaba。spring-cloud版本、springboot版本、spring-cloud-alibaba版本见项目配置：

```xml
<springcloud.version>Finchley.SR2</springcloud.version>
<springboot.version>2.0.6.RELEASE</springboot.version>
<springcloud.alibaba.version>0.2.2.RELEASE</springcloud.alibaba.version>
```

### producer

pom依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--nacos注册中心依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--通过spring-cloud-stream的方式集成rocketmq的依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
    </dependency>

    <!--用于监控-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

创建一个binding用于发送消息到broker，里面定义几个管道，每个管道用来发送消息到不同的topic。并且在程序入口类上通过注解启用binding。

```java
public interface MySource {
    @Output("output1")
    MessageChannel output1();

    @Output("output2")
    MessageChannel output2();

    @Output("output3")
    MessageChannel output3();
}
```

```java
@SpringBootApplication
@EnableBinding({MySource.class})
public class RocketMQProducerApplication {
    public static void main(String[] args) {
        SpringApplication.run(RocketMQProducerApplication.class,args);
    }
}
```

定义一个消息实体模型

```java
public class Foo {

    private int id;
    private String bar;

    //getter & setter...
}
```

定义一个发送消息的服务，这个服务中使用上面定义的binding来发送消息，`sendTransactionalMsg`这个方法是专门用于发送事务消息的。

```java
@Service
public class SenderService {

    @Autowired
    private MySource source;

    public void send(String msg) {
        source.output1().send(MessageBuilder.withPayload(msg).build());
    }

    public <T> void sendWithTags(T msg, String tag) {
        Message message = MessageBuilder.createMessage(msg, new MessageHeaders(Stream.of(tag).collect(Collectors.toMap(str -> MessageConst.PROPERTY_TAGS, String::toString))));
        source.output1().send(message);
    }

    public <T> void sendObject(T msg, String tag) {
        Message message = MessageBuilder.withPayload(msg)
                .setHeader(MessageConst.PROPERTY_TAGS, tag)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build();
        source.output1().send(message);
    }

    public <T> void sendTransactionalMsg(T msg, int num) {
        MessageBuilder builder = MessageBuilder.withPayload(msg).setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON);
        builder.setHeader("test", String.valueOf(num));
        builder.setHeader(RocketMQHeaders.TAGS, "binder");
        Message message = builder.build();
        source.output2().send(message);
    }

}
```

为了测试事务消息，定义一个本地事务监听器，用来保证本地事务执行完成之后消息正确发送到broker。关于rocketmq的事务消息，参考[这里](https://qigangzhong.github.io/2019/09/09/rocketmq/#%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF)。

```java
@RocketMQTransactionListener(txProducerGroup = "myTxProducerGroup", corePoolSize = 5, maximumPoolSize = 10)
public class TransactionListenerImpl implements RocketMQLocalTransactionListener {
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        Object num = msg.getHeaders().get("test");

        if ("1".equals(num)) {
            System.out.println("executer: " + new String((byte[]) msg.getPayload()) + " unknown");
            return RocketMQLocalTransactionState.UNKNOWN;
        }
        else if ("2".equals(num)) {
            System.out.println("executer: " + new String((byte[]) msg.getPayload()) + " rollback");
            return RocketMQLocalTransactionState.ROLLBACK;
        }
        System.out.println("executer: " + new String((byte[]) msg.getPayload()) + " commit");
        return RocketMQLocalTransactionState.COMMIT;
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        System.out.println("check: " + new String((byte[]) msg.getPayload()));
        return RocketMQLocalTransactionState.COMMIT;
    }
}
```

我们通过创建一个controller，在浏览器调用接口的方式来发送消息到broker，这里测试发送3种消息，第一种是通过`SenderService`来发送带tag或不带tag的消息，第二种是直接通过binding来发送消息，第三种是发送事务消息。

```java
@RestController
public class SendMsgController {
    @Autowired
    private SenderService senderService;
    @Autowired
    private MySource mySource;

    @GetMapping("/send1")
    public String send1(){
        int count = 5;
        for (int index = 1; index <= count; index++) {
            String msgContent = "msg-" + index;
            if (index % 3 == 0) {
                senderService.send(msgContent);
            }
            else if (index % 3 == 1) {
                senderService.sendWithTags(msgContent, "tagStr");
            }
            else {
                senderService.sendObject(new Foo(index, "foo"), "tagObj");
            }
        }

        return "ok";
    }

    @GetMapping("/send3")
    public String send3(){
        int count = 20;
        for (int index = 1; index <= count; index++) {
            String msgContent = "pullMsg-" + index;
            mySource.output3().send(MessageBuilder.withPayload(msgContent).build());
        }

        return "ok";
    }

    @GetMapping("/sendTx")
    public String sendTx(){
        // COMMIT_MESSAGE message
        senderService.sendTransactionalMsg("transactional-msg1", 1);
        // ROLLBACK_MESSAGE message
        senderService.sendTransactionalMsg("transactional-msg2", 2);
        // ROLLBACK_MESSAGE message
        senderService.sendTransactionalMsg("transactional-msg3", 3);
        // COMMIT_MESSAGE message
        senderService.sendTransactionalMsg("transactional-msg4", 4);

        return "ok";
    }
}
```

最重要的配置文件，以配置的方式指定发送消息的binding中的管道对应的目标topic信息

```bash
spring.application.name=rocketmq-producer-example
server.port=28081

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

spring.cloud.nacos.discovery.server-addr=192.168.237.128:8848



spring.cloud.stream.rocketmq.binder.name-server=192.168.237.128:9876

spring.cloud.stream.bindings.output1.destination=test-topic
spring.cloud.stream.bindings.output1.content-type=application/json
spring.cloud.stream.rocketmq.bindings.output1.producer.group=binder-group
spring.cloud.stream.rocketmq.bindings.output1.producer.sync=true

spring.cloud.stream.bindings.output2.destination=TransactionTopic
spring.cloud.stream.bindings.output2.content-type=application/json
spring.cloud.stream.rocketmq.bindings.output2.producer.transactional=true
spring.cloud.stream.rocketmq.bindings.output2.producer.group=myTxProducerGroup

spring.cloud.stream.bindings.output3.destination=pull-topic
spring.cloud.stream.bindings.output3.content-type=text/plain
spring.cloud.stream.rocketmq.bindings.output3.producer.group=pull-binder-group
```

### consumer

pom依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--nacos注册中心依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--通过spring-cloud-stream的方式集成rocketmq的依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
    </dependency>

    <!--用于监控-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

定义一个binding，用来接收broker的消息，里面定义了不同的管道来接收不同topic的消息。在程序启动类上通过注解启用binding。

```java
public interface MySink {

    @Input("input1")
    SubscribableChannel input1();

    @Input("input2")
    SubscribableChannel input2();

    @Input("input3")
    SubscribableChannel input3();

    @Input("input4")
    SubscribableChannel input4();

    /**
     * pull模式的代码如果放开会报错，原因是：
     * pull模式在alibaba分支的spring-cloud-stream-binder-rocketmq中才支持，而且只有v2.0.0.RELEASE之后版本支持
     * 测试项目中使用的是apache版本的spring-cloud-stream-binder-rocketmq，版本是0.2.2.RELEASE（针对Finchley的版本）是不支持pull模式的
     * 目前最新版本0.9.0.RELEASE（针对Greenwitch的版本）也是不支持的
     */
    /*@Input("input5")
    PollableMessageSource input5();*/
}
```

```java
@SpringBootApplication
@EnableBinding({ MySink.class })
public class RocketMQConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(RocketMQConsumerApplication.class,args);
    }
}
```

同样要定义一个接收实体消息的模型，跟producer发送的模型一样

```java
public class Foo {

    private int id;
    private String bar;
    //getters & setters...
}
```

创建一个服务，来监听topic中的消息，主要通过`@StreamListener`注解来指定监听哪个topic

```java
@Service
public class ReceiveService {

    @StreamListener("input1")
    public void receiveInput1(String receiveMsg) {
        System.out.println("input1 receive: " + receiveMsg);
    }

    @StreamListener("input2")
    public void receiveInput2(String receiveMsg) {
        System.out.println("input2 receive: " + receiveMsg);
    }

    @StreamListener("input3")
    public void receiveInput3(@Payload Foo foo) {
        System.out.println("input3 receive: " + foo);
    }

    @StreamListener("input4")
    public void receiveTransactionalMsg(String transactionMsg) {
        System.out.println("input4 receive transaction msg: " + transactionMsg);
    }
}
```

配置中指定需要监听的topic信息

```bash
spring.application.name=rocketmq-consumer-example
server.port=28082
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

spring.cloud.nacos.discovery.server-addr=192.168.237.128:8848

spring.cloud.stream.rocketmq.binder.name-server=192.168.237.128:9876

spring.cloud.stream.bindings.input1.destination=test-topic
spring.cloud.stream.bindings.input1.content-type=text/plain
spring.cloud.stream.bindings.input1.group=test-group1
spring.cloud.stream.rocketmq.bindings.input1.consumer.orderly=true

spring.cloud.stream.bindings.input2.destination=test-topic
spring.cloud.stream.bindings.input2.content-type=text/plain
spring.cloud.stream.bindings.input2.group=test-group2
spring.cloud.stream.rocketmq.bindings.input2.consumer.orderly=false
spring.cloud.stream.rocketmq.bindings.input2.consumer.tags=tagStr
spring.cloud.stream.bindings.input2.consumer.concurrency=20
spring.cloud.stream.bindings.input2.consumer.maxAttempts=1

spring.cloud.stream.bindings.input3.destination=test-topic
spring.cloud.stream.bindings.input3.content-type=application/json
spring.cloud.stream.bindings.input3.group=test-group3
spring.cloud.stream.rocketmq.bindings.input3.consumer.tags=tagObj
spring.cloud.stream.bindings.input3.consumer.concurrency=20

spring.cloud.stream.bindings.input4.destination=TransactionTopic
spring.cloud.stream.bindings.input4.content-type=text/plain
spring.cloud.stream.bindings.input4.group=transaction-group
spring.cloud.stream.bindings.input4.consumer.concurrency=5

spring.cloud.stream.bindings.input5.destination=pull-topic
spring.cloud.stream.bindings.input5.content-type=text/plain
spring.cloud.stream.bindings.input5.group=pull-topic-group
```

### 测试

1. 首先启动nacos注册中心

2. 启动rocketmq

3. 启动consumer、producer

4. 调用producer的发送消息接口

```bash
curl http://localhost:28081/send1
curl http://localhost:28081/send3
curl http://localhost:28081/sendTx
```

## 参考

[官方spring-cloud-alibaba的rocketmq-example](https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-examples/rocketmq-example/readme-zh.md)
