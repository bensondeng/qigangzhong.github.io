---
layout: post
title:  "vagrant virtualbox"
categories: tools
tags:  vagrant virtualbox
author: 网络
---

* content
{:toc}









## 安装

[virtualbox 5.1.24 download](http://download.virtualbox.org/virtualbox)

[vagrant 1.9.7 download](https://releases.hashicorp.com/vagrant/)

[vagrant box centos7 download](http://www.vagrantbox.es/)


```bash
# 加载box
vagrant box add centos7 centos-7.0-x86_64.box
# 查看box列表
vagrant box list
# 在需要生成虚拟机的目录下面进行box初始化
vagrant init centos7
# 启动box
vagrant up
# 通过ssh客户端连接
127.0.0.1:2222
vagrant/vagrant
```
## 参考

[vagrant cloud](https://app.vagrantup.com/boxes/search)
