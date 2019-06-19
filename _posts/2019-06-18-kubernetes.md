---
layout: post
title:  "kubernetes"
categories: docker
tags:  docker kubernetes
author: 刚子
---

* content
{:toc}









## 一、 概念

* Cluster
    > kubernetes用一种更高效的方式自动化集群中的应用容器的发布和调度
  * Master
    > 负责管理集群
  * Node
    > 每个node上都有一个kubelet，用来管理pod、node，并且通过`Kubernetes API`与master通信，生产环境建议最少3个node。每个node也必须具有一个容器运行时（例如Docker）。一个Node可以包含多个Pod
  * Pod
    > 一组紧密关联的容器以及共享资源（存储、网络、镜像版本等其它信息）的逻辑分组
  * Service
    > 虽然每个pod都有独立的网络ip地址，但都是不对外公开的，需要通过service来暴露地址供外部访问

cluster->deployment->service->node->pod->container

![cluster与node关系](/images/docker/cluster.svg)

![node与pod的关系](/images/docker/nodes.svg)

![service与label](/images/docker/service_label.svg)

## 二、安装

## 三、Quick Start

### 发布应用

```bash
minikube version
# 通过minikube创建一个cluster
minikube start
kubectl version
kubectl cluster-info
# 新建的cluster默认有一个node(类型是master)
kubectl get nodes

# 根据镜像创建一个deployment
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080

# 获取pod名称
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

kubectl get deployments

```

### pod及容器操作

```bash
# 获取pod信息
kubectl get pods
# 获取pod描述信息（node，pods(ip，端口，镜像，卷，等等)，deployments）
kubectl describe pods
# 新开terminal执行proxy（pod是在一个独立、私有网络里面运行，所以需要使用proxy与之交互）
kubectl proxy
# 回到第一个terminal执行version查看版本
curl http://localhost:8001/version
# 获取pod名称
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
# 查看pod中的容器应用的输出
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
# 查看pod中容器的日志（pod中只有一个容器时可以不指定容器名称）
kubectl logs $POD_NAME
# 对容器执行命令，查看环境变量
kubectl exec $POD_NAME env
# 进入到容器里面
kubectl exec -ti $POD_NAME bash
```

### 通过service来暴露容器里面的应用端口

```bash
# 获取pod信息
kubectl get pods
# 获取service信息
kubectl get services
# 创建service，暴露pod
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
# 查看service暴露到哪个端口
kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
# 通过外网访问暴露出来的端口
curl $(minikube ip):$NODE_PORT

# 根据label来查询pod
kubectl get pods -l run=kubernetes-bootcamp
# 根据label来查询service
kubectl get services -l run=kubernetes-bootcamp
# 给pod设置label
kubectl label pod $POD_NAME app=v1
# 查询pod的基础信息，看一下label是否更新
kubectl describe pods $POD_NAME
# 根据这个新的label来查询pod信息
kubectl get pods -l app=v1

# 删除service
kubectl delete service -l run=kubernetes-bootcamp
# 删除service之后从外部已无法访问容器
curl $(minikube ip):$NODE_PORT
# 但是在pod里面还是可以内部访问容器的
kubectl exec -ti $POD_NAME curl localhost:8080
```

### 应用水平扩展

```bash
# 查看当前发布的应用有几个节点（READY列：当前运行副本数量/总副本数量）
kubectl get deployment
# 添加副本数量，扩展为4个副本，pod数量会从1扩展为4
kubectl scale deployments/kubernetes-bootcamp --replicas=4
# 查看pod，每个pod有独立的ip
kubectl get pods -o wide
# 副本数量修改的动作会被记录到deployment的事件中，可以查看到
kubectl describe deployments/kubernetes-bootcamp

# 暴露service，查看一下对外暴露的ip和端口
kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
# 访问一下对外暴露的ip及端口，多次访问之后可以发现请求落在了不同的pod上面，service对后端的pod自动做了负载均衡
curl $(minikube ip):$NODE_PORT
```

### 应用水平收缩

```bash
# 收缩到2个副本，pod数量会从4变为2
kubectl scale deployments/kubernetes-bootcamp --replicas=2
kubectl get deployments
kubectl get pods -o wide
```

### 应用镜像升级、回滚

```bash
# 升级之前先查看一下当前的pod列表
kubectl get deployments
kubectl get pods
# 通过set image命令升级镜像版本
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
# 再次查看pod列表会发现老的pod全部被停止，但没有删除，新启动了新的pod
kubectl get pods

# 通过service查看当前对外暴露的ip端口
kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
# 访问服务查看是否已被更新为新版本应用
curl $(minikube ip):$NODE_PORT
# 查看升级状态
kubectl rollout status deployments/kubernetes-bootcamp
# 通过pod的描述信息可以看到pod的事件日志
kubectl describe pods

# 操作一次失败的升级，镜像不存在
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
# 查看状态
kubectl get deployments
kubectl get pods
kubectl describe pods
# 执行回滚版本的操作
kubectl rollout undo deployments/kubernetes-bootcamp
# 查看pod信息看是否回滚到上个版本
kubectl get pods
kubectl describe pods
```

## 参考

[kubernetes官方教程](https://kubernetes.io/docs/tutorials/hello-minikube/)
