---
layout: post
title:  "centos中安装jenkins"
categories: tools
tags:  jenkins
author: 网络
---

* content
{:toc}

总结centos7.4上安装jenkins、shadowsocks客户端的步骤











## 一、安装jenkins

> 直接安装好一点，通过docker来启动翻墙麻烦

```bash
# 1. 安装jdk
yum install -y java
# 2. 添加jenkins yum库，并通过yum安装jekins
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
yum install -y jenkins

-----------------
# 直接下载安装包的方式
wget http://pkg.jenkins-ci.org/redhat-stable/jenkins-2.7.3-1.1.noarch.rpm
rpm -ivh jenkins-2.7.3-1.1.noarch.rpm
-----------------

# 3. 修改配置(启动端口默认值JENKINS_PORT="8080")
vi /etc/sysconfig/jenkins
# 4. 启动、查看状态、停止
service jenkins start/status/stop/restart
# 5. 查看初始密码
cat /var/lib/jenkins/secrets/initialAdminPassword
# 查看日志，或者可以直接访问以下链接来查看日志：http://{{host}}:8080/log/all
tail -f /var/log/jenkins/jenkins.log
```

## 二、安装shadowsocks客户端

```bash
# 1. 安装epel扩展源和pip
yum -y install epel-release
yum -y install python-pip
# 2. 安装Shadowsocks客户端
pip install shadowsocks
# 3. 创建配置文件
mkdir /etc/shadowsocks
vi /etc/shadowsocks/shadowsocks.json
---------------------------------------------------------------
{
    "server":"10.10.10.10",
    "server_port":11211,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"password",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}
---------------------------------------------------------------
# 4. 配置自启动
vim /etc/systemd/system/shadowsocks.service
---------------------------------------------------------------
[Unit]
Description=Shadowsocks
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/sslocal -c /etc/shadowsocks/shadowsocks.json
[Install]
WantedBy=multi-user.target
---------------------------------------------------------------
systemctl enable shadowsocks.service
systemctl start shadowsocks.service
systemctl status shadowsocks.service
# 5. 验证Shadowsocks客户端是否正常运行，返回结果应该是shadowsocks服务器地址
curl --socks5 127.0.0.1:1080 http://httpbin.org/ip

# 6. 安装配置Privoxy，使用Privoxy将http、https流量转发到shadowsocks本地代理端口上
sudo yum -y install privoxy
systemctl enable privoxy
systemctl start privoxy
systemctl status privoxy
# 7. 配置Privoxy
vim /etc/privoxy/config
---------------------------------------------------------------
listen-address 127.0.0.1:8118 # 8118 是默认端口，不用改
forward-socks5t / 127.0.0.1:1080 . #转发到本地端口
---------------------------------------------------------------
# 8. 设置http/https代理
vim /etc/profile
---------------------------------------------------------------
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
---------------------------------------------------------------
source /etc/profile
# 9. 测试是否生效
curl www.google.com
```

## 参考

[centos下搭建Jenkins持续集成环境](https://www.cnblogs.com/loveyouyou616/p/8714544.html)

[CentOS 7安装配置Shadowsocks客户端](http://ju.outofmemory.cn/entry/357236)

[Privoxy教程](https://www.cnblogs.com/hongdada/p/10787924.html)
