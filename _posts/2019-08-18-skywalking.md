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

## 安装backend和UI

从[skywalking-downloads](http://skywalking.apache.org/downloads/)页面下载6.3最新版本的linux安装包，放置到centos7服务器上，根据[Quick Start](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-ui-setup.md)来启动后台管理界面。

```bash
cd /root/sw
tar -zxvf apache-skywalking-apm-6.3.0.tar.gz
cd bin
sh ./startup.sh
```

> 默认的配置：
> * Storage, use H2 by default, in order to make sure, don't need further deployment.
> * Backend listens 0.0.0.0/11800 for gRPC APIs and 0.0.0.0/12800 for http rest APIs. In Java, .NetCore, Node.js, Istio agents/probe, set the gRPC service address to ip/host:11800. (ip/host is where the backend at)
> * UI listens 8080 port and request 127.0.0.1/12800 to do GraphQL query.

## 参考

[skywalking-6.3.0 doc](https://github.com/apache/skywalking/blob/v6.3.0/docs/README.md)

[downloads](http://skywalking.apache.org/downloads/)

[online demo](http://122.112.182.72:8080/)

[java-agent](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/service-agent/java-agent/README.md)

[backend-ui-setup](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-ui-setup.md)

[生产部署backend-ui](https://github.com/apache/skywalking/blob/v6.3.0/docs/en/setup/backend/backend-ui-setup.md#deploy-backend-and-ui)
