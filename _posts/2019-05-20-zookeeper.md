---
layout: post
title:  "zookeeper"
categories: tools
tags:  zookeeper
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
1. 单台机器上创建3个不同的配置文件zoo1.cfg、zoo2.cfg、zoo3.cfg， 配置内容见下方。

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
#强制删除节点，即使节点下有子节点
rmr /dubbo
```

## 二、zk临时节点及会话

* 临时节点

* zookeeper会话

> 客户端和服务端建立连接后，会话随之建立，生成一个全局唯一的会话ID(Session ID)。服务器和客户端之间维持的是一个长连接，在SESSION_TIMEOUT时间内，服务器会确定客户端是否正常连接(客户端会定时向服务器发送heart_beat，服务器重置下次SESSION_TIMEOUT时间)。因此，在正常情况下，Session一直有效，并且ZK集群所有机器上都保存这个Session信息。
> zookeeper服务端配置文件zoo.cfg中可以配置两个个参数来设置会话超时时间范围，在创建客户端ZkClient的时候都可以指定会话超时时间，但是这个时间必须在服务器zoo.cfg中配置的超时时间范围内，会话超时之后，会话中创建的临时节点、watcher都会被删除。

```bash
#zoo.cfg会话超时时间范围配置
#默认情况下zoo.cfg中的配置ticktime=2000，单位是ms
#minSessionTimeout默认值=2*ticktime
#maxSessionTimeout默认值=20*ticktime
minSessionTimeout=2000
maxSessionTimeout=5000
```

ZkClient创建客户端的参数：  
zkServer, zookeeper服务器地址，用逗号分隔  
sessionTimeout，会话超时时间，单位毫秒，默认为30000ms  
connectionTimeout，连接超时时间  
IZkConnection，连接的实现类  
zkSerializer，自定义序列化实现

## 三、zookeeper java客户端

[示例代码，配置中心，注册中心](https://gitee.com/qigangzhong/toolset/tree/master/zk-demo)

### 配置中心实现原理

![zk_config_management.git](/images/zk/zk_config_management.gif)

### 分布式锁实现原理

![zk_lock.gif](/images/zk/zk_lock.gif)

### 注册中心实现原理

## 参考

[ZooKeeper学习](https://www.cnblogs.com/wuxl360/p/5817471.html)

[基于zookeeper的配置管理中心](https://blog.csdn.net/flykinghg/article/details/52934693)

[zookeeper-step配置中心，分布式锁，分布式session代码示例](https://github.com/flykingmz/zookeeper-step)

[采用zookeeper的EPHEMERAL节点机制实现服务集群的陷阱](https://yq.aliyun.com/articles/227260)

[zkclient sessiontimeout session 会话](https://blog.csdn.net/xiaoliuliu2050/article/details/82501226)

[*****zookeeper 典型应用场景](https://blog.csdn.net/xiaoliuliu2050/article/details/54951529)