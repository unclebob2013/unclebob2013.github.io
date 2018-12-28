---
title: kubernetes总结
date: 2018-12-28 10:25:17
tags:
- kubernetes
categories: kubernetes
top: true
copyright: true
---
# kubernetes 总结
## 基础
### 0x01 kubernetes包含几个组件。各个组件的功能是什么。组件之间是如何交互的
### Master

#### kube-apiserver
暴露整个集群的API，任何资源请求/调用操作都是通过调用API来实现，唯一一个跟ETCD通信的组件

#### ETCD
存储整个集群的信息

#### kube-scheduler
负责对于新建pod的调度，基于node节点的软件硬件负载和亲和性来选择节点
 
#### kube-controller-manager
运行在master节点的控制器的集合
- Node Controller 监控和相应节点down
- Replication Controller 为每个RC对象维护正确数量的pod
- Endpoints Controller 负责建立Endpoints，链接Service和Pods
- Service Account & Token Controllers 为新的Namespace创建默认账户和API访问Token

#### cloud-controller-manager
负责和底层云服务交互(可选)
- Node Controller
- Route Controller
- Service Controller
- Volume Controller

<!-- more -->

### Node

#### kubelet
运行与每个节点，根据下发的PodSpecs确保Pod中的容器的运行和健康

#### kube-proxy
维护主机上kube服务涉及的网络规则和连接转发，负责service的实现,采用iptables(Nat/Filter)
- 侦听service更新事件，并更新service相关的iptables规则
- 侦听endpoint更新事件，更新endpoint相关的iptables规则（如 KUBE-SVC-链中的规则），然后将包请求转入endpoint对应的Pod

#### Container Runtime
容器运行时，支持Docker rkt runc 


### Addon

#### DNS
集群的DNS，负责内部服务的DNS解析

#### dashboard

#### Container Resource Monitoring
容器资源监控

#### Cluster-level Logging

---

### 0x02 k8s的pause容器有什么用。是否可以去掉。

pause容器是pod级别的，每个pod中的pause容器为业务容器的父容器
- 共享命名空间 IPC Hostname Network PID
- 而是提供Pid Namespace并使用init进程。



---

### 0x03 k8s中的pod内几个容器之间的关系是什么
都由Pod中的pause容器创建，共享network和volume资源

---

### 0x04 一个经典pod的完整生命周期。
- Pending：Pod 定义正确，提交到 Master，但其包含的容器镜像还未完全创建。通常处在 Master 对 Pod 的调度过程中。
- ContainerCreating：Pod 的调度完成，被分配到指定 Node 上。处于容器创建的过程中。通常是在拉取镜像的过程中。
- Running：Pod 包含的所有容器都已经成功创建，并且成功运行起来。
- Successed：Pod 中所有容器都成功结束，且不会被重启。这是 Pod 的一种最终状态。
- Failed：Pod 中所有容器都结束，但至少一个容器以失败状态结束。这也是 Pod 的一种最终状态。

---

### k8s的service和ep是如何关联和相互影响的
Service 通过selector选择pod,生成endpoint，并通过endpoint和后端pod通信
kube-proxy
- 监听到service被删除，则删除和该service同名的endpoint对象
- 监听到新的service被创建，则根据新建service信息获取相关pod列表，然后创建对应endpoint对象
- 监听到service被更新，则根据更新后的service信息获取相关pod列表，然后更新对应endpoint对象
- 监听到pod事件，则更新对应的service的endpoint对象，将podIp记录到endpoint中

### 详述kube-proxy原理
[kube-proxy iptables](https://blog.csdn.net/zhangxiangui40542/article/details/79486995)

iptables - netfilter - NAT/FILTER

访问 nodePort - clusterIp:port - Pod:port
Kubernetes通过在目标node的iptables中的nat表的PREROUTING和POSTROUTING链中创建一系列的自定义链 （这些自定义链主要是“KUBE-SERVICES”链、“KUBE-POSTROUTING”链、每个服务所对应的“KUBE-SVC-XXXXXXXXXXXXXXXX”链和“KUBE-SEP-XXXXXXXXXXXXXXXX”链），然后通过这些自定义链对流经到该node的数据包做DNAT和SNAT操作以实现路由、负载均衡和地址转换。

iptables -S -t nat

### rc/rs功能是怎么实现的。详述从API接收到一个创建rc/rs的请求，到最终在节点上创建pod的全过程，尽可能详细。另外，当一个pod失效时，kubernetes是如何发现并重启另一个pod的？

[Deployment](https://jimmysong.io/kubernetes-handbook/concepts/deployment.html)

而当前推荐的做法是使用Deployment+ReplicaSet
RC是通过ReplicationManager监控RC和RC内Pod的状态，从而增删Pod，以实现维持特定副本数的功能。


### deployment/rs有什么区别。其使用方式、使用条件和原理是什么。

deployment控制rs，rs控制pod
deployment是rs的超集，提供更多的部署功能，如：回滚、暂停和重启、 版本记录、事件和状态查看、滚动升级和替换升级。

### group中的cpu有哪几种限制方式。k8s是如何使用实现request和limit的。

cpu.cfs_quota_us、cpu.cfs_period_us 限制CPU的使用时间
cpu.shares 限制group的配额

request 容器使用最小资源
limit 容器使用的最大资源
[k8s中资源使用](https://mp.weixin.qq.com/s/rj2DmHmRofJ1FvVmD6kMUw)

---

## 拓展实践

### 设想一个一千台物理机，上万规模的容器的kubernetes集群，请详述使用kubernetes时需要注意哪些问题？应该怎样解决？（提示可以从高可用，高性能等方向，覆盖到从镜像中心到kubernetes各个组件等）

[集群规模庞大时要做得限制](https://kubernetes.io/zh/docs/admin/cluster-large/)

配额问题针对云厂商
存储在独立的etcd集群中
管理节点和组件的规格
插件资源占用限制

### 一千到五千台的瓶颈

[京东大规模集群实现](https://juejin.im/entry/5b6d55d2e51d4519475f91c8)

[1-5000](https://www.goodrain.com/2018/04/24/tech-20180424/)

### kubernetes的运行中有哪些注意的要点。

### 集群发生雪崩的条件，以及预防手段。

[京东大规模k8s集群的精细化运营](https://dbaplus.cn/news-141-2139-1.html)

### 设计一种可以替代kube-proxy的实现

### sidecar的设计模式如何在k8s中进行应用。有什么意义。
[sidecar模式](http://www.servicemesher.com/blog/sidecar-design-pattern-in-microservices-ecosystem/)
将应用程序的功能划分为单独的进程可以被视为 Sidecar 模式
- 通过抽象出与功能相关的共同基础设施到一个不同层降低了微服务代码的复杂度。
- 因为你不再需要编写相同的第三方组件配置文件和代码，所以能够降低微服务架构中的代码重复度。
- 降低应用程序代码和底层平台的耦合度。

### 灰度发布是什么。如何使用k8s现有的资源实现灰度发布。
[介绍](https://blog.csdn.net/wangyinghong_2013/article/details/78650290)
[使用istio实现k8s的灰度发布](https://zhaohuabing.com/2017/11/08/istio-canary-release/)

### 介绍k8s实践中踩过的比较大的一个坑和解决方式。




