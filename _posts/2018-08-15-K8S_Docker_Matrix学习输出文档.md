---
layout: post
title: K8S_Docker_Matrix学习输出文档
categories: container
description: K8S_Docker_Matrix学习输出文档
keywords: Kubernetes, docker
---

# K8S
## K8S整体架构
k8s集群节点分为master和node节点，其整体架构图如下：  
![](/images/kubernetes/20171208223632158.jpeg)  
### master节点的工作流程图如下：  
![](/images/kubernetes/20170530215659358.png)  

1. Kubecfg将特定的请求，比如创建Pod，发送给Kubernetes Client。

2. Kubernetes Client将请求发送给API server。

3. API Server根据请求的类型，比如创建Pod时storage类型是pods，然后依此选择何种REST Storage API对请求作出处理。

4. REST Storage API对的请求作相应的处理。

5. 将处理的结果存入高可用键值存储系统Etcd中。

6. 在API Server响应Kubecfg的请求后，Scheduler会根据Kubernetes Client获取集群中运行Pod及Minion/Node信息。

7. 依据从Kubernetes Client获取的信息，Scheduler将未分发的Pod分发到可用的Minion/Node节点上。  


### node节点结构图如下：  
![](/images/kubernetes/20170530215731801.png)  

## K8S关键源码
