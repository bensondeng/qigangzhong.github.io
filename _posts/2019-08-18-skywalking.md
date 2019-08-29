---
layout: post
title:  "skywalking"
categories: tools
tags:  skywalking
author: 网络
---

* content
{:toc}

总结skywalking的安装及使用方法







## 概念

skywalking是一款为微服务准备的国产APM（application performance management）产品，它使用java-agent探针无侵入收集应用的追踪信息发送给后台服务backend-service，提供服务性能报表，调用链路追踪，告警等功能。作者[吴晟](https://github.com/wu-sheng/me)，目前该项目已加入apache孵化，已经有很多国内公司接入。

![skywalking_arch](/images/arch/skywalking_arch.png)

skywalking backend-ui站点上面的几个常见概念：

* Avg SLA： 服务可用性（主要是通过请求成功与失败次数来计算）
* CPM： 每分钟调用次数
* Avg Response Time： 平均响应时间

## 安装backend和backend-ui

从[skywalking-downloads](http://skywalking.apache.org/downloads/)页面下载6.3最新版本的linux安装包，放置到centos7服务器上，根据[Quick Start](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-ui-setup.md)来启动后台管理界面。

```bash
cd /usr/local/skywalking
tar -zxvf apache-skywalking-apm-6.3.0.tar.gz
cd cd apache-skywalking-apm-bin/
# startup.sh会同时启动backend以及backend-ui
sh ./bin/startup.sh

# 【注意】backend-ui上面显示的数据是根据时间来的，要保证服务器的时区正确才行
# 设置服务器时区为上海
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

> 默认的配置：
>
> * Storage, use H2 by default, in order to make sure, don't need further deployment. 在`config/application.yml`中进行配置。
> * Backend listens 0.0.0.0/11800 for gRPC APIs and 0.0.0.0/12800 for http rest APIs. In Java, .NetCore, Node.js, Istio agents/probe, set the gRPC service address to ip/host:11800. (ip/host is where the backend at). 在`config/application.yml`中进行配置。
> * UI listens 8080 port and request 127.0.0.1/12800 to do GraphQL query. 在`webapp/webapp.yml`中进行配置。

打开`http://host:port/`就可以访问到UI界面，端口就是`webapp/webapp.yml`中配置的端口。

### 集成ES，日志定期清理

TODO

## 使用java-agent

### 集成探针到项目中

安装包下面有一个目录agent，里面是探针应用，将整个agent目录copy到服务器上的固定目录，或者放到项目中，修改`agent/config/agent.config`配置文件，里面有两个比较重要的配置

```bash
# 在backend-ui上显示的应用名称
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}
# backend的gRPC调用地址，探针收集到数据后通过这个地址将数据传送给backend
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:192.168.237.128:11800}
```

在项目VM启动参数中添加探针相关参数

```bash
# 通过skywalking.agent.service_name、skywalking.collector.backend_service可以重写agent/config/ageng.config里面对应的配置信息，通过这种方式，多个应用可以共用一份agent而不用到处copy
java -javaagent:/path/to/skywalking-agent/skywalking-agent.jar -Dskywalking.agent.service_name=my-service -Dskywalking.collector.backend_service=localhost:11800 -jar my-service.jar
```

在IDE中测试时，可以直接指定VM参数如下：

```bash
# 这里指定service_name，backend_service的地址使用agent.config配置文件中的
-javaagent:"E:\gitee\springcloud.f\skywalking\agent\skywalking-agent.jar" -Dskywalking.agent.service_name=sw-service-hi-client
```

### 告警

[告警](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/master/setup/backend/backend-alarm.html)主要是通过`config/alarm-settings.yml`中的配置驱动的，配置中分为两部分，一个是rule，代表告警规则，一个是webhooks代表告警消息推送url，这个url接收参数必须符合固定格式。

这个告警功能需要自己定制，比如短信、邮件、钉钉、微信，都需要自己提供一个固定的api入口供backend-service推送告警消息。而且需要更新告警规则的时候是不是需要重启服务？

## 其它

### 性能

参考官方性能测试项目[skywalking agent performance test](https://skyapmtest.github.io/Agent-Benchmarks/README_zh.html)，还有一篇[官方发布的对比文章](APM巅峰对决：skywalking P.K. Pinpoint)可以参考。

### 同类产品对比

| 类别                 | Zipkin                                     | Pinpoint             | SkyWalking           | CAT                                |
| ---------------------- | ------------------------------------------ | -------------------- | -------------------- | ---------------------------------- |
| 实现方式           | 拦截请求，发送（HTTP，mq）数据至zipkin服务 | java探针，字节码增强 | java探针，字节码增强 | 代码埋点（拦截器，注解，过滤器等） |
| 接入方式           | 基于linkerd或者sleuth方式，引入配置即可 | javaagent字节码   | javaagent字节码   | 代码侵入                       |
| 客户端语言支持       | java,c#,go,php等 | java、php   | Java, .NET Core, NodeJS and PHP   | Java, C/C++, Node.js, Python, Go 等       |
| agent到collector的协议 | http,MQ                                    | thrift               | gRPC                 | http/tcp                           |
| OpenTracing            | √                                        | ×                   | √                  | ×                                 |
| 颗粒度              | 接口级                                  | 方法级            | 方法级            | 代码级                          |
| 全局调用统计     | ×                                         | √                  | √                  | √                                |
| traceid查询          | √                                        | ×                   | √                  | ×                                 |
| 报警                 | ×                                         | √                  | √                  | √                                |
| JVM监控              | ×                                         | ×                   | √                  | √                                |
| 健壮度              | **                                         | *****                | ****                 | *****                              |
| 数据存储           | ES，mysql,Cassandra,内存                | Hbase                | ES，H2，MySql，TiDB，Sharding-Sphere      | mysql,hdfs                         |
{: .table.table-bordered }

上面的表格信息有些地方不太准确，大体上反映了各个产品的特点。

skywalking总体体验下来只适合应用链路监控，没有提供业务相关的监控以及其它扩展功能，比较而言，[CAT](https://github.com/dianping/cat)虽然侵入性比较强，但是功能更强大，可以实现应用链路监控，同时也可以实现业务数据监控，并且提供了强大的图表功能展示监控数据，告警功能也非常丰富。

## 参考

[skywalking-6.3.0 doc](https://github.com/apache/skywalking/blob/v6.3.0/docs/README.md)

[downloads](http://skywalking.apache.org/downloads/)

[online demo](http://122.112.182.72:8080/)

[java-agent](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/service-agent/java-agent/README.md)

[backend-ui-setup](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/ui-setup.md)

[backend-setup](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-setup.md)

[backend-cluster-nacos](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-cluster.md#nacos)

[backend-storage](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-storage.md)

[生产部署backend-ui](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-ui-setup.md#deploy-backend-and-ui)

[中文文档](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/master/)

[SkyWalking 客户端配置](https://www.jianshu.com/p/77b4e70c7817)
