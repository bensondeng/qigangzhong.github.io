---
layout: post
title:  "通过supervisor管理应用进程"
categories: tools
tags:  supervisor
author: 刚子
---

* content
{:toc}


## 前言


总结通过supervisor管理应用进程的知识点

##  课程目录











## 一、安装、卸载

* 安装

```
>yum install python-setuptools
>easy_install pip
>pip install supervisor  -- 或者-- easy_install supervisor
>mkdir /etc/supervisor/

#安装完成之后，在/etc/supervisor目录下生成配置文件
>echo_supervisord_conf>/etc/supervisor/supervisord.conf
```

* 卸载

```
>pip uninstall supervisor
```

## 二、修改配置文件

```
>vi /etc/supervisor/supervisord.conf

#把末尾的include去掉；添加配置文件
[include]
files = /etc/supervisor/conf.d/*.conf

#开启管理界面
[inet_http_server]
port=*:9001        ;127.0.0.1:9001需要修改为*:9001或者0.0.0.0:9001
```

添加每个应用独立的配置文件，供supervisor加载

```
>vi /etc/supervisor/conf.d/test.conf

#应用配置文件内容
[program:springboot.test]
command=/bin/bash -c "/root/java/jdk1.8.0_191/bin/java -jar springboot.test-1.0.0.jar"
directory=/var/www/test
autorestart=false #进程死亡后是否自动拉起来
stderr_logfile=/var/log/test/err.log
stdout_logfile=/var/log/test/out.log
environment=ASPNETCORE_ENVIRONMENT=Development
user=root
stopsignal=INT
```

## 三、启动

```
>supervisord -c /etc/supervisor/supervisord.conf
```

设置开机启动

```
>vi /etc/rc.d/sh/test.sh

#!/bin/bash
# 开机启动supervisor
supervisord -c /etc/supervisor/supervisord.conf

#然后赋予执行权限
>chmod +x /etc/rc.d/sh/test.sh
```

```
>vi /etc/rc.d/rc.local

>touch /var/lock/subsys/local

#开机启动supervisor脚本
/etc/rc.d/sh/test.sh

#赋予执行权限
>chmod +x /etc/rc.d/rc.local
```

## 四、常用命令

* 启动

```
supervisord -c /etc/supervisor/supervisord.conf
```

* 停止

```
supervisorctl shutdown
```

* 更新新的配置到supervisord

```
supervisorctl update
```

* 重新启动配置中的所有程序

```
supervisorctl reload
```

* 启动某个进程(program_name=你配置中写的程序名称)

```
supervisorctl start program_name
```

* 查看正在守护的进程

```
supervisorctl
```

* 停止某一进程 (program_name=你配置中写的程序名称)

```
pervisorctl stop program_name
```

* 重启某一进程 (program_name=你配置中写的程序名称)

```
supervisorctl restart program_name
```

* 停止全部进程

```
supervisorctl stop all
```

## 参考
