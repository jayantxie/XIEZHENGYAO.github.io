---
layout: post
title: 初识Kubernetes源码(1)—源码结构分析
categories: container
description: 初识Kubernetes源码(1)—源码结构分析
keywords: Kubernetes, docker
---

&nbsp;&nbsp;出于工作需要，在不久前刚刚接触Kubernetes，不得不说这对我是一个非常大的挑战。一来博主此前较少接触golang开发，二来也对基于golang的K8s和docker等容器技术了解甚浅。  
&nbsp;&nbsp;两个月的工作过程中，只有大约一周左右的时间在初步了解K8s的架构，以及和公司自身的容器平台之间的关联逻辑，而后的大部分工作都游离在表面，知其然而不知其所以然。这样的状态，身为一个有点自尊心的程序员是无法忍受的，因此，我下定决心在未来的几个月内深挖K8s的源码实现，本篇将拉开序幕。

## 从目录讲起
说到源码，必须先上GitHub链接，找到golang工程目录，git clone下方地址：  
[https://github.com/kubernetes/kubernetes.git](https://github.com/kubernetes/kubernetes.git)  
![](/images/kubernetes/source_code_struct.png)  
*kubernetes源码目录结构*  
其中，  
|目录名 | 说明 |
| ------ | ----- |
|`api` | 输出接口文档用，基本是json源码 |
|`build`| 存放构建脚本 |
|`cmd` | 所有的二进制可执行文件入口代码，即各种命令的接口代码 |
|`pkg`| 项目diamante主目录，cmd只是接口，这里是具体实现。|
|`plugin`|插件|
|`test`|存放测试相关的工具|
|`third_party`|存放第三方工具|
|`docs`|第三方工具|
|`Godeps`|项目依赖的Go的第三方包，比如docker客户端sdk，rest等|
|`hack`|工具箱，存放各种编译、构建和校验的脚本|

我们主要关注cmd包和pkg包，cmd包可以视作各组件的启动入口，pkg包则包含各模块的具体实现和算法。K8s的主要几个模块都包含在内，我们可以在下一节中初步理解各模块的作用及相互间的关联。

## K8s整体架构
K8s集群节点分为master和node节点，其整体架构图如下：  
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

> 从上图中，我们总结出K8s在master节点上有如下组件：  
**API Server**：资源操作入口，提供RESTFul接口供外部客户或内部组件调用，封装后端实现，保证集群操作的安全；  
**Controller Manager**：内部管理控制中心，主要分为Endpoint Controller和Replication Controller。前者定期关联service和pod(关联信息由endpoint对象维护)，后者控制pod数总与其定义的pod数保持一致；  
**Scheduler**：集群分发调度器，负责对pod即将部署的node打分，同时检测pod和node信息，部署pod之后将pod与node关联信息写回API Server；  
**Kubectl**：集群管理命令行工具集，通过客户端的kubectl命令集操作，API Server响应对应的命令结果，从而达到对kubernetes集群的管理。


### node节点结构图如下：  
![](/images/kubernetes/20170530215731801.png)  

> 可以看到在node节点上有如下组件：  
**Kubelet**：node节点上的pod管理员，主要负责：  
&nbsp;&nbsp;1. 负责Node节点上pod的创建、修改、监控、删除等全生命周期的管理；  
&nbsp;&nbsp;2. 定时上报本Node的状态信息给API Server；  
&nbsp;&nbsp;3. kubelet是Master API Server和Node之间的桥梁，接收Master API Server分配给它的commands和work，通过kube-apiserver间接与Etcd集群交互，读取配置信息；  
**Proxy**：负载均衡，路由转发；