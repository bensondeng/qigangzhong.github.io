---
layout: post
title:  "zookeeper"
categories: tools
tags:  zk,zookeeper,tools
author: 刚子
---

* content
{:toc}

总结zk知识点

## 一、安装配置

### 1. 下载解压

[官方下载地址](http://zookeeper.apache.org/releases.html#download)

```bash
tar -zxvf zookeeper-3.4.13.tar.gz
```

### 2. 配置

* copy示例配置文件

```bash
cd conf
cp zoo_sample.cfg zoo.cfg
```

* 设置环境变量

```bash
vim /etc/profile

export ZOOKEEPER_HOME=/root/zookeeper/zookeeper-3.4.13
export PATH=.:$PATH:$ZOOKEEPER_HOME/bin

. /etc/profile
```

* 配置伪集群环境

```
1. 单台机器上创建3个不同的配置文件zoo1.cfg、zoo2.cfg、zoo3.cfg， 配置内容见右方。

2. 在配置文件指定的dataDir目录下面创建myid文件，内容分别为0,1,2，对应配置文件中的server.X

3. 分别创建dataDir/dataLogDir指定的目录

4. 启动3个服务：
sh ./bin/zkServer.sh start ./conf/zoo1.cfg
sh ./bin/zkServer.sh start ./conf/zoo2.cfg
sh ./bin/zkServer.sh start ./conf/zoo3.cfg

5. 查看3个服务的状态：
sh ./bin/zkServer.sh status ./conf/zoo1.cfg
sh ./bin/zkServer.sh status ./conf/zoo2.cfg
sh ./bin/zkServer.sh status ./conf/zoo3.cfg
```

```cfg
# zoo1.cfg
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/usr/local/zk/data_1
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

dataLogDir=/usr/local/zk/logs_1

server.0=localhost:2287:3387
server.1=localhost:2288:3388
server.2=localhost:2289:3389
```

```cfg
# zoo2.cfg
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/usr/local/zk/data_2
# the port at which the clients will connect
clientPort=2182
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

dataLogDir=/usr/local/zk/logs_2

server.0=localhost:2287:3387
server.1=localhost:2288:3388
server.2=localhost:2289:3389
```

```cfg
# zoo3.cfg
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/usr/local/zk/data_3
# the port at which the clients will connect
clientPort=2183
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

dataLogDir=/usr/local/zk/logs_3

server.0=localhost:2287:3387
server.1=localhost:2288:3388
server.2=localhost:2289:3389
```

### 3. 启动zk服务、自带客户端使用

* 启动/停止zk服务，查看zk服务状态

```bash
sh ./bin/zkServer.sh start
sh ./bin/zkServer.sh stop
sh ./bin/zkServer.sh status
```

* 自带客户端使用

```bash
zkCli.sh -server localhost:2181
create /zk myData
ls /
get /zk
set /zk myData1
delete /zk
```

## 参考

