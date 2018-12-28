---
title: kubernetes架构
comments: true
top: false
copyright: true
date: 2018-12-28 17:44:27
categories:
- kubernetes
tags:
- kubernetes
photos:
link:
description:
---

# k8s架构

节选自 **https://draveness.me/understanding-kubernetes** 请大家到原网站观看更多内容

Kubernetes 遵循非常传统的客户端服务端架构，客户端通过 RESTful 接口或者直接使用 kubectl 与 Kubernetes 集群进行通信，这两者在实际上并没有太多的区别，后者也只是对 Kubernetes 提供的 RESTful API 进行封装并提供出来。

![KUBERNETES ARCHITECTURE](https://img.draveness.me/2018-11-25-kubernetes-architecture.png)

每一个 Kubernetes 就集群都由一组 Master 节点和一系列的 Worker 节点组成，其中 Master 节点主要负责存储集群的状态并为 Kubernetes 对象分配和调度资源。

<!--more-->

# Master

作为管理集群状态的 Master 节点，它主要负责接收客户端的请求，安排容器的执行并且运行控制循环，将集群的状态向目标状态进行迁移，Master 节点内部由三个组件构成：

![KUBERNETES MASTER NODE](https://img.draveness.me/2018-11-25-kubernetes-master-node.png)

其中 API Server 负责处理来自用户的请求，其主要作用就是对外提供 RESTful 的接口，包括用于查看集群状态的读请求以及改变集群状态的写请求，也是唯一一个与 etcd 集群通信的组件。

而 Controller 管理器运行了一系列的控制器进程，这些进程会按照用户的期望状态在后台不断地调节整个集群中的对象，当服务的状态发生了改变，控制器就会发现这个改变并且开始向目标状态迁移。

最后的 Scheduler 调度器其实为 Kubernetes 中运行的 Pod 选择部署的 Worker 节点，它会根据用户的需要选择最能满足请求的节点来运行 Pod，它会在每次需要调度 Pod 时执行。

# Worker

其他的 Worker 节点实现就相对比较简单了，它主要由 kubelet 和 kube-proxy 两部分组成：

![KUBERNETES WORKER NODE](https://img.draveness.me/2018-11-25-kubernetes-worker-node.png)

kubelet 是一个节点上的主要服务，它周期性地从 API Server 接受新的或者修改的 Pod 规范并且保证节点上的 Pod 和其中容器的正常运行，还会保证节点会向目标状态迁移，该节点仍然会向 Master 节点发送宿主机的健康状况。

另一个运行在各个节点上的代理服务 kube-proxy 负责宿主机的子网管理，同时也能将服务暴露给外部，其原理就是在多个隔离的网络中把请求转发给正确的 Pod 或者容器。
