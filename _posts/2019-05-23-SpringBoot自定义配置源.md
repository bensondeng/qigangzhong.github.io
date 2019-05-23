---
layout: post
title: "SpringBoot自定义配置源(支持自动刷新)"
categories: SpringBoot
tags: SpringBoot config
author: 刚子
---

* content
{:toc}

介绍SpringBoot除去本地配置文件以外，如何加入自定义的配置源，通过@Value注入配置，并且能够定时刷新最新配置信息











## 一、EnvironmentPostProcessor

[示例代码](https://gitee.com/qigangzhong/share.demo/tree/master/environment-post-processor-demo)

## 二、PropertySourceLocator

[示例代码](https://gitee.com/qigangzhong/share.demo/tree/master/property-source-locator-demo)

## 三、Archaius PolledConfigurationSource

[示例代码](https://gitee.com/qigangzhong/share.demo/tree/master/springboot-archaius-demo)

## 参考
