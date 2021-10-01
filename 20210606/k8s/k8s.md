
[TOC]

# Kubernetes概述

docker和k8s是云原生时代两颗璀璨的明珠，分别对应了云计算中的两大核心功能：虚拟化和调度。资源的虚拟化为调度提供了前提，而调度则增加了系统了灵活性和资源的利用率。

kubernetes,简称k8s，至今已经已经成为容器编排的事实标准。本文则主要讲述k8s相关的一概念，并对其余的云上调度算法进行比较。

## k8s的功能

k8s作为云原生时代容器编排的事实标准，自然具有其丰富的功能，概括起来主要有如下功能：

- 服务发现与负载均衡
  - 通过DNS名称或者IP地址访问容器
  - 分配节点间的网络流量
- 存储编排
- 自动部署和回滚
- 自动完成装箱计算
- 自我修复
- 密钥与配置管理

## k8s的架构

![image-20210606090903746](https://gitee.com/weixiao619/pic/raw/master/image-20210606090903746.png)

如图为k8s的架构，其中一个Node一般对应一个宿主机，Node中实际运行pod，pod中包含有一个或多个container(Docker)，这些contianer之间共享着网络和存储资源。以一个实际的web服务为例，实际提供服务能力的是pod ,而Service 从逻辑上定义了运行在集群中的一组 Pod对外提供相同的服务。

每一个Node上各运行一个kubelet和kube-proxy，其中kubelet负责确保容器在pod上的运行，而kube-proxy负责处理节点的网络代理。

而workload(工作负载)是对pod的描述，以deployment为例，deployment通过控制replicaset来间接控制pod

### ReplicationController

k8s的的副本控制逻辑，k8s中并不区分创建和回滚的操作，以deployment为例，更新deployment时，如果与历史中的某个版本相同，则执行回滚操作。

![image-20210625192255148](https://gitee.com/weixiao619/pic/raw/master/image-20210625192255148.png)



### workload的类型

k8s的工作负载有不同的类型，各个类型都有其特点。

#### Deployment

Deployment是最常见的一种工作负载类型，是原有ReplicasSet的升级版，适合管理无状态应用，即多实例之间没有区别

#### StatefulSet

StatefulSet不同于Deployment类型，主要用于有状态应用的，有状态应用各个pod之间会有所区别，同时也可以将各个pod与持久化存储相对应

#### DeamonSet

DeamonSet一般是Node为单位运行的pod管理器，常见的情况是可以用DeamonSet来管理收集日志的pod,确保可以采集到每一个Node的文件

#### Jobs

上述的三种工作负载类型都是用于管理服务的，默认情况下一直处于运行状态；Job类型则较为不同，主要管理任务类型的实例，任务完成后实例就会关闭；CronJob与Job类似，区别在于普通的Job是一次性的任务，CronJob会根据时间规划反复运行。AI中的训练任务就是典型的Job类型，不过原生的k8s管理器对Job类型支持的并不是很好。不过得益于k8s优异的可拓展性，社区有很多优化项目既包括对Job类型工作负载本身的优化，也包括对Job类型任务特殊的调度逻辑设计。

## k8s的组件

![Kubernetes 组件](https://gitee.com/weixiao619/pic/raw/master/components-of-kubernetes-20210605222057314.svg)

### 控制面板组件

- Kube-apiserver

  对外暴露k8s的服务

- etcd

  etcd是高可用的k-v数据库，在k8s中作为集群运行所需的持久化数据存储组件

- Kube-scheduler

  调度组件，选择将pod调度到哪个git当中

- Kube-controller-manager

  从逻辑上讲，每个[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。

  这些控制器包括:

  - 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应
  - 任务控制器（Job controller）: 监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
  - 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)
  - 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

- Cloud-controller-manager

  云控制器管理器是指嵌入特定云的控制逻辑的 [控制平面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane)组件。 云控制器管理器允许您链接聚合到云提供商的应用编程接口中， 并分离出相互作用的组件与您的集群交互的组件。

### Node组件

- kubelet
  - 运行在集群的每一个节点上，确保容器在pod上的运行
- kube-proxy
  - 运行在每一个节点上的网络代理，负责节点和pod间通讯的网路规则
- Container Runtime
  - Docker,containerd,CRI-O等任何实现k8s CRI (容器运行环境接口)
  - 最新k8s已不支持docker

### 插件

- DNS
- Web界面
- 容器资源监控
- 集群日志



## k8s的调度

k8s作为云原生的编排工具，调度自然是其核心所在，在 k8s 中，*调度* 是指将 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 放置到合适的 [Node](https://kubernetes.io/zh/docs/concepts/architecture/nodes/) 上，然后由对应 Node 上的 [Kubelet](https://kubernetes.io/docs/reference/generated/kubelet) 运行这些 pod。

### 调度器的触发

在了解具体的调度逻辑之前，首先介绍一下k8s的一个调度事件是如何产生和处理的。

 ![image-20210607191247355](https://gitee.com/weixiao619/pic/raw/master/image-20210607191247355.png)

如上图所示为pod的调度过程: 

1. reflector通过List的Watch方式监听k8s的API，从而获取事件
2. 将获取到的事件对象存入 Delta 的FIFO队列中
3. Informer从Delta队列中获取事件对象
4. Informer首先传递给Indexer
5. Indexer负责调用存储组件以k-v的形式存储下来，其中的key为该对象的`<namespace> / <name>`组合
6. 随后，Informer将事件发送到资源事件处理器，
7. 资源事件的Handler将事件放入工作队列workqueue中
8. 事件处理器获取到事件的key，并交给具体的Handle
9. Handle通过Indexer的引用，根据key拿到存储的事件对象，进行相应处理

#####  组件

在上述的调度过程中，提到了k8s的三个组件，下面是各个组件功能的简要介绍

1.`reflector`:用来watch特定的k8s API资源。具体的实现是通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源。当reflector通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的Delta Fifo队列中。

2.`Informer`：informer从Delta Fifo队列中弹出对象。执行此操作的功能是processLoop。base controller的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。

3.`Indexer`：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 在Store中定义了一个名为`MetaNamespaceKeyFunc`的默认函数，该函数生成对象的键作为该对象的`<namespace> / <name>`组合

- `Informer reference`：指的是Informer实例的引用，定义如何使用自定义资源对象。 自定义控制器代码需要创建对应的Informer。
- `Indexer reference`: 自定义控制器对Indexer实例的引用。自定义控制器需要创建对应的Indexser。

> client-go中提供`NewIndexerInformer`函数可以创建Informer 和 Indexer。

- `Resource Event Handlers`：资源事件回调函数，当它想要将对象传递给控制器时，它将被调用。 编写这些函数的典型模式是获取调度对象的key，并将该key排入工作队列以进行进一步处理。
- `Workqueue`：任务队列。 编写资源事件处理程序函数以提取传递的对象的key并将其添加到任务队列。
- `Process Item`：处理任务队列中对象的函数， 这些函数通常使用Indexer引用或Listing包装器来重试与该key对应的对象。

### 调度的逻辑

上述流程的最后触发的Handle即为调度的具体逻辑，调度的过程由**Kube-scheduler**组件来负责，k8s为了保证调度的效率和调度流程的简洁性，将调度大致分为了两个阶段，分别为**过滤**和**打分**。对应的过程如下：

![](https://gitee.com/weixiao619/pic/raw/master/640-20210607192255383.png)



过滤阶段将所有满足 Pod 调度需求的 Node 选出来。例如，PodFitsResources 过滤函数会检查候选 Node 的可用资源能否满足 Pod 的资源请求。在过滤之后，得出一个 Node 列表，里面包含了所有可调度节点；通常情况下，这个 Node 列表包含不止一个 Node。如果这个列表是空的，代表这个 Pod 不可调度；

打分阶段根据过滤阶段得到的Node列表，为每一个可调度节点进行打分。最后，**kube-scheduler** 会将 Pod 调度到得分最高的 Node 上。如果存在多个得分最高的 Node，kube-scheduler 会从中随机选取一个。

k8s将调度过程分为

下面分别对两个流程进行介绍

### 1.过滤

过滤过程中，会过滤出可调度的Node节点，过滤策略有如下几种：

#### 1.1. GeneralPredicates

负责最基础的调度策略，过滤宿主机的CPU和内存等资源信息

#### 1.2. Volume过滤规则

负责与容器持久化Volume相关的调度策略。例如：

检查多个 Pod 声明挂载的持久化 Volume 是否有冲突；

检查一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目；

检查Pod 对应的 PV 的 nodeAffinity 字段，是否跟某个节点的标签相匹配

#### 1.3. 检查调度Pod是否满足Node本身的条件

如PodToleratesNodeTaints负责检查的就是我们前面经常用到的 Node 的“污点”机制。NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足。

#### 1.4. affinity and anti-affinity

根据pod的node之间的亲和性与反亲和性进行过滤

### 2. 打分

```go
//筛选函数
// scheduleOne does the entire scheduling workflow for a single pod.
scheduleOne()

// 1. 超时打日志


// 2. 预选策略选出满足调度条件的Node节点
findNodesThatFit()
// Run "prefilter" plugins.
	s := fwk.RunPreFilterPlugins(ctx, state, pod)

// "NominatedNodeName" can potentially be set in a previous scheduling cycle as a result of preemption.
	// This node is likely the only candidate that will fit the pod, and hence we try it first before iterating over all nodes.
feasibleNodes, err := g.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)


// 用不同的过滤插件{NodePort, Volume, Label等}进行过滤
runFilterPlugin()

// 3. 对节点进行打分
prioritizeNodes()

// 运行预打分插件
// InterPodAffinity, NodeAffinity, PodTopologySpread, 
RunPreScorePlugins()

// 运行打分插件
RunScorePlugins()


```

![image-20210602215451585](https://gitee.com/weixiao619/pic/raw/master/image-20210602215451585.png)

### 代码逻辑

![img](https://gitee.com/weixiao619/pic/raw/master/1599387171_7_w1514_h1290.png)

1. 通过sched.NextPod()函数从优先队列中获取一个优先级最高的待调度Pod资源对象，如果没有获取到，那么该方法会阻塞住；
2. 通过sched.Algorithm.Schedule调度函数执行Predicates的调度算法与Priorities算法，挑选出一个合适的节点；
3. 当没有找到合适的节点时，调度器会尝试调用prof.RunPostFilterPlugins抢占低优先级的Pod资源对象的节点；
4. 当调度器为Pod资源对象选择了一个合适的节点时，通过sched.bind函数将合适的节点与Pod资源对象绑定在一起；

## k8s的服务发现

 **Service 从逻辑上定义了运行在集群中的一组 Pod，通过标签进行绑定**，这些 Pod 提供了相同的功能。 当每个 Service 创建时，会被分配一个唯一的 IP 地址（也称为 clusterIP）。 这个 IP 地址与一个 Service 的生命周期绑定在一起，当 Service 存在的时候它也不会改变。 可以配置 Pod 使它与 Service 进行通信，Pod 知道与 Service 通信将被自动地负载均衡到该 Service 中的某些 Pod 上。

### 暴露服务的方式

- NodePort
- LoadBalancer
- Ingress

### headless服务

headless服务，即无头服务，是一种特殊的服务类型，默认情况下，k8s的DNS会按照自己的负载均衡策略将域名转换为随机的后端podIp,这在不依赖k8s来进行负载均衡的情况下，会有一些问题。因此，k8s支持将服务声明为Headless，此时，k8s的DNS会返回域名绑定的所有podIp,由用户自己进行负载均衡。

### 拓扑感知提示

#### EndpointSlice 控制器

此特性开启后，EndpointSlice 控制器负责在 EndpointSlice 上设置提示信息。 控制器按比例给每个区域分配一定比例数量的端点。 这个比例来源于此区域中运行节点的 [可分配](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) CPU 核心数。 例如，如果一个区域拥有 2 CPU 核心，而另一个区域只有 1 CPU 核心， 那控制器将给那个有 2 CPU 的区域分配两倍数量的端点。

### 服务内部流量策略

在相同node中访问pod

### 虚拟IP与Service代理

#### userspace 代理模式

![userspace 代理模式下 Service 概览图](https://gitee.com/weixiao619/pic/raw/master/services-userspace-overview.svg)



#### iptable代理模式

![Services overview diagram for iptables proxy](https://gitee.com/weixiao619/pic/raw/master/services-iptables-overview.svg)

#### ipvs代理模式

![IPVS代理的 Services 概述图](https://gitee.com/weixiao619/pic/raw/master/services-ipvs-overview.svg)

## k8s的拓展性

![kubernetes-extensions](https://img.draveness.me/kubernetes-extensions-2021-03-10-16153845432607.png)

k8s通过定义接口的方式，解耦了许多底层组件，极大的增强了k8s的可扩展性。其中网络和容器的通过接口实现可拓展性的方式在上文讲述容器运行时的部分有更为详细的阐述。

上图中从上到下依次为

1. CRD (CustomResourceDefinition)，定制化资源定义
2. CNI (ContainerNetworkInterface)，容器网络接口
3. CRI (ContainerRuntimeInterface)，容器运行时接口
4. CSI (ContainerStorageInterface)，容器存储接口
5. Scheduling Framework. 调度框架，



## k8s的局限性

虽然k8s是云原生时代事实上的容器编排标准，但并不代表其完美无缺，这里对k8s现存的一些问题和一些解决方案的探索做一个简要的阐述。

### 集群管理

#### 水平伸缩性

首先是几圈

Kubernetes 社区对外宣传的是单个集群最多支持 5,000 节点，Pod 总数不超过 150,000，容器总数不超过 300,000 以及单节点 Pod 数量不超过 100 个，与几万节点的 Apache Mesos 集群、50,000 节点的微软 YARN 集群相比，Kubernetes 的集群规模整整差了一个数量级。虽然阿里云的工程师也通过优化 Kubernetes 的各个组件实现了 5 位数的集群规模，但是与其他的资源管理方式相比却有比较大的差距。

### 多集群管理

k8s的多集群管理会带来 资源不平衡、跨集群访问困难等问题

#### 现有的一些探索方案

##### kubefed

kubefed(Kubernetes Cluster Federation)是k8s社区给出的解决方案，提供了跨集群资源和网络管理的功能

![concepts.png](https://gitee.com/weixiao619/pic/raw/master/concepts.png)



### 应用场景

#### 应用分发

Kubernetes 主项目提供了几种部署应用的最基本方式，分别是 `Deployment`、`StatefulSet` 和 `DaemonSet`，这些资源分别适用于无状态服务、有状态服务和节点上的守护进程，这些资源能够提供最基本的策略，但是它们无法处理更加复杂的应用

很多常见的场景，例如只运行一次的 DaemonSe以及金丝雀和蓝绿部署等功能，现在的资源也存在很多问题，例如 StatefulSet 在初始化容器中卡住无法回滚和更新。

#### 批处理调度

机器学习、批处理任务和流式任务等工作负载的运行从 Kubernetes 诞生第一天起到今天都不是它的强项，大多数的公司都会使用 Kubernetes 运行在线服务处理用户请求，用 Yarn 管理的集群运行批处理的负载。当然在批处理任务方面，k8s社区本身也做了部分改进，有kube-batch的插件主要用于优化批处理任务的调度，同时华为在kube-batch的基础上演进出Volcano调度器有效的提高了k8s集群中运行批处理作业的能力，获得了广泛的认可。在后续比较云原生调度器的内容中会进一步对Volcano调度器做更为详细的阐述。

#### 硬多租户

今天的 Kubernetes 还很难做到硬多租户支持，也就是同一个集群的多个租户不会相互影响，也感知不到彼此的存在。尽管目前k8s用命名空间来划分虚拟集群，但也很难实现很好的资源隔离。



# 以下为旧版，忽略



## k8s提供的功能

## k8s的组件

![Kubernetes 组件](https://gitee.com/weixiao619/pic/raw/master/components-of-kubernetes-20210605222057314.svg)

### 控制面板组件

- Kube-apiserver

  对外暴露k8s的服务

- etcd

  etcd是高可用的k-v数据库，在k8s中作为集群运行所需的持久化数据存储组件

- Kube-scheduler

  调度组件，选择将pod调度到哪个git当中

- Kube-controller-manager

  从逻辑上讲，每个[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。

  这些控制器包括:

  - 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应
  - 任务控制器（Job controller）: 监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
  - 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)
  - 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

- Cloud-controller-manager

  云控制器管理器是指嵌入特定云的控制逻辑的 [控制平面](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-control-plane)组件。 云控制器管理器允许您链接聚合到云提供商的应用编程接口中， 并分离出相互作用的组件与您的集群交互的组件。

### Node组件

- kubelet
  - 运行在集群的每一个节点上，确保容器在pod上的运行
- kube-proxy
  - 运行在每一个节点上的网络代理，负责节点和pod间通讯的网路规则
- Container Runtime
  - Docker,containerd,CRI-O等任何实现k8s CRI (容器运行环境接口)
  - 最新k8s已不支持docker

### 插件

- DNS
- Web界面
- 容器资源监控
- 集群日志

## k8s暴露服务的架构

 ![image-20210607191247355](https://gitee.com/weixiao619/pic/raw/master/image-20210607191247355.png)

#### pod的调度过程：

1. reflector通过List的Watch方式监听k8s的API
2. 将获取到的事件对象存入 Delta 的FIFO队列中
3. Informer从Delta队列中获取事件对象
4. Informer首先传递给Indexer
5. Indexer负责调用存储组件以k-v的形式存储下来，其中的key为该对象的`<namespace> / <name>`组合
6. 随后，Informer将事件发送到资源事件处理器，
7. 资源事件的Handler将事件放入工作队列workqueue中
8. 事件处理器获取到事件的key，并交给具体的Handle
9. Handle通过Indexer的引用，根据key拿到存储的事件对象，进行相应处理

#####  组件

1.`reflector`:用来watch特定的k8s API资源。具体的实现是通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源。当reflector通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的Delta Fifo队列中。

2.`Informer`：informer从Delta Fifo队列中弹出对象。执行此操作的功能是processLoop。base controller的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。

3.`Indexer`：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 在Store中定义了一个名为`MetaNamespaceKeyFunc`的默认函数，该函数生成对象的键作为该对象的`<namespace> / <name>`组合



- `Informer reference`：指的是Informer实例的引用，定义如何使用自定义资源对象。 自定义控制器代码需要创建对应的Informer。
- `Indexer reference`: 自定义控制器对Indexer实例的引用。自定义控制器需要创建对应的Indexser。

> client-go中提供`NewIndexerInformer`函数可以创建Informer 和 Indexer。

- `Resource Event Handlers`：资源事件回调函数，当它想要将对象传递给控制器时，它将被调用。 编写这些函数的典型模式是获取调度对象的key，并将该key排入工作队列以进行进一步处理。
- `Workqueue`：任务队列。 编写资源事件处理程序函数以提取传递的对象的key并将其添加到任务队列。
- `Process Item`：处理任务队列中对象的函数， 这些函数通常使用Indexer引用或Listing包装器来重试与该key对应的对象。



具体的调度过程则如下图所示

![](https://gitee.com/weixiao619/pic/raw/master/640-20210607192255383.png)





## Pod



pod是k8s管理的的最小计算单元

一个pod中包含一个或多个container,共享存储和网络资源

Pod也可以包含init container，同时也有[ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)可以用于debug

Pod之间的上下文共享，依赖一系列linux的隔离技术实现，如 [namespace](https://en.wikipedia.org/wiki/Linux_namespaces)、cgroups等

pod并不需要直接创建，而是通过deployment或statefulset等进行创建

pod的更新并不是更新已有的pod，而是创建一个新的pod去替换已有的pod

### pod内共享资源和通信

#### 存储

一个pod中共享一系列存储卷，pod中所有的containers都可以访问这些共享的卷，从而实现共享存储。pod内部的

#### 网络

- 每个pod被指定一个唯一的ip地址,pod中的每个容器都共享ip地址和端口。

- pod内部的容器之间可以通过**localhost**进行通信，也可以通过linux标准的进程间通讯方式进行通讯

容器的特权模式：允许容器使用操作系统的管理员能力，例如操作网络栈，访问硬件资源等

#### 静态pod

静态pod是直接通过kubelet进行管理的特殊节点，不能通过APIServer暴露，

### pod的限制性拓扑策略

pod的拓扑策略配置样例，主要依赖四个参数：

```yaml
#pod.spec.topologySpreadConstraints demo
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  topologySpreadConstraints:
    - maxSkew: <integer>   #标识节点的最大深度差
      topologyKey: <string>  # 标识一组调度节点
      whenUnsatisfiable: <string> # 不满足时的调度策略
      labelSelector: <object> # 用于寻找labels匹配的的pods
```



### pod如何管理多个containers

Pod 被设计成支持形成内聚服务单元的多个协作过程（形式为容器）。 Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。 容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身。

例如，你可能有一个容器，为共享卷中的文件提供 Web 服务器支持，以及一个单独的 “sidecar”容器负责从远端更新这些文件，如下图所示：

![example pod diagram](https://gitee.com/weixiao619/pic/raw/master/pod.svg)

你可以使用工作负载资源来创建和管理多个 Pod。 资源的控制器能够处理副本的管理、上线，并在 Pod 失效时提供自愈能力。 例如，如果一个节点失败，控制器注意到该节点上的 Pod 已经停止工作， 就可以创建替换性的 Pod。调度器会将替身 Pod 调度到一个健康的节点执行



### 示例

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```

## 工作负载类型

### deploymnet

### statefulset

### DeamonSet

### Jobs



## 服务

 **Service 从逻辑上定义了运行在集群中的一组 Pod，通过标签进行绑定**，这些 Pod 提供了相同的功能。 当每个 Service 创建时，会被分配一个唯一的 IP 地址（也称为 clusterIP）。 这个 IP 地址与一个 Service 的生命周期绑定在一起，当 Service 存在的时候它也不会改变。 可以配置 Pod 使它与 Service 进行通信，Pod 知道与 Service 通信将被自动地负载均衡到该 Service 中的某些 Pod 上。

### 暴露服务的方式

- NodePort
- LoadBalancer
- Ingress

### 拓扑感知提示

#### EndpointSlice 控制器

此特性开启后，EndpointSlice 控制器负责在 EndpointSlice 上设置提示信息。 控制器按比例给每个区域分配一定比例数量的端点。 这个比例来源于此区域中运行节点的 [可分配](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) CPU 核心数。 例如，如果一个区域拥有 2 CPU 核心，而另一个区域只有 1 CPU 核心， 那控制器将给那个有 2 CPU 的区域分配两倍数量的端点。

### 服务内部流量策略

在相同node中访问pod

### 虚拟IP与Service代理

#### userspace 代理模式

![userspace 代理模式下 Service 概览图](https://gitee.com/weixiao619/pic/raw/master/services-userspace-overview.svg)



#### iptable代理模式

![Services overview diagram for iptables proxy](https://gitee.com/weixiao619/pic/raw/master/services-iptables-overview.svg)

#### ipvs代理模式

![IPVS代理的 Services 概述图](https://gitee.com/weixiao619/pic/raw/master/services-ipvs-overview.svg)





## 调度

在 Kubernetes 中，*调度* 是指将 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 放置到合适的 [Node](https://kubernetes.io/zh/docs/concepts/architecture/nodes/) 上，然后对应 Node 上的 [Kubelet](https://kubernetes.io/docs/reference/generated/kubelet) 才能够运行这些 pod。

Kube-scheduler给一个pod做调度分为两部分内容：

1.过滤

2.打分

### 过滤

过滤过程中，会过滤出可调度的Node节点，过滤策略有如下几种：

#### GeneralPredicates

负责最基础的调度策略，过滤宿主机的CPU和内存等资源信息

#### Volume过滤规则

负责与容器持久化Volume相关的调度策略。例如：

检查多个 Pod 声明挂载的持久化 Volume 是否有冲突；

检查一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目；

检查Pod 对应的 PV 的 nodeAffinity 字段，是否跟某个节点的标签相匹配

#### 检查调度Pod是否满足Node本身的条件

如PodToleratesNodeTaints负责检查的就是我们前面经常用到的 Node 的“污点”机制。NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足。

#### affinity and anti-affinity

根据pod的node之间的亲和性与反亲和性进行过滤

### 打分

```go
//筛选函数
// scheduleOne does the entire scheduling workflow for a single pod.
scheduleOne()

// 1. 超时打日志


// 2. 预选策略选出满足调度条件的Node节点
findNodesThatFit()
// Run "prefilter" plugins.
	s := fwk.RunPreFilterPlugins(ctx, state, pod)

// "NominatedNodeName" can potentially be set in a previous scheduling cycle as a result of preemption.
	// This node is likely the only candidate that will fit the pod, and hence we try it first before iterating over all nodes.
feasibleNodes, err := g.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)


// 用不同的过滤插件{NodePort, Volume, Label等}进行过滤
runFilterPlugin()

// 3. 对节点进行打分
prioritizeNodes()

// 运行预打分插件
// InterPodAffinity, NodeAffinity, PodTopologySpread, 
RunPreScorePlugins()

// 运行打分插件
RunScorePlugins()


```

![image-20210602215451585](https://gitee.com/weixiao619/pic/raw/master/image-20210602215451585.png)

### 代码逻辑

![img](https://gitee.com/weixiao619/pic/raw/master/1599387171_7_w1514_h1290.png)

1. 通过sched.NextPod()函数从优先队列中获取一个优先级最高的待调度Pod资源对象，如果没有获取到，那么该方法会阻塞住；
2. 通过sched.Algorithm.Schedule调度函数执行Predicates的调度算法与Priorities算法，挑选出一个合适的节点；
3. 当没有找到合适的节点时，调度器会尝试调用prof.RunPostFilterPlugins抢占低优先级的Pod资源对象的节点；
4. 当调度器为Pod资源对象选择了一个合适的节点时，通过sched.bind函数将合适的节点与Pod资源对象绑定在一起；

## kubelet工作原理

集群的每个Node均有一个kubelet进程，pod被调度到特定的Node后，由kubelet进行管理。

### pod管理

Kubelet 以 PodSpec 的方式工作。PodSpec 是描述一个 Pod 的 YAML 或 JSON 对象。 kubelet 采用一组通过各种机制提供的 PodSpecs（主要通过 apiserver），并确保这些 PodSpecs 中描述的 Pod 正常健康运行

### 容器监控和健康检查

- LivenessProbe
- ReadinessProbe

### 容器运行时

#### 工作模式

kubelet通过容器运行时接口 (Container Runtime Interface，CRI)与容器进行交互。

CRI将kubelet与容器运行时解耦：

![image-20210603161339399](https://gitee.com/weixiao619/pic/raw/master/image-20210603161339399.png)

#### 架构

kubelet通过CRI结构与实际的container runtime进行交互，解耦不同容器的实现细节。如下为容器运行时的具体架构：

![image-20210603161320110](https://gitee.com/weixiao619/pic/raw/master/image-20210603161320110.png)

容器运行时主要定义了CRI Server和Streaming Server两个服务，其中CRI Server实现了具体的CRI服务接口，由Runtime Service和 Image Manger Service两个服务组成，具体的实现可参考下文中的源码分析。

CNI: Container Network Interface 容器网络接口

通过CNI将容器运行时与底层的网络插件实现解耦，使得容器无需关心底层的网络细节。

#### 交互

![image-20210605201715782](https://gitee.com/weixiao619/pic/raw/master/image-20210605201715782.png)

如图所示为一个exec操作的典型交互逻辑

#### CRI接口定义源码

```go

// ImageManagerService 需要实现的功能
type ImageManagerService interface {
	// ListImages lists the existing images.
	ListImages(filter *runtimeapi.ImageFilter) ([]*runtimeapi.Image, error)
	// ImageStatus returns the status of the image.
	ImageStatus(image *runtimeapi.ImageSpec) (*runtimeapi.Image, error)
	// PullImage pulls an image with the authentication config.
	PullImage(image *runtimeapi.ImageSpec, auth *runtimeapi.AuthConfig, podSandboxConfig *runtimeapi.PodSandboxConfig) (string, error)
	// RemoveImage removes the image.
	RemoveImage(image *runtimeapi.ImageSpec) error
	// ImageFsInfo returns information of the filesystem that is used to store images.
	ImageFsInfo() ([]*runtimeapi.FilesystemUsage, error)
}

// RuntimeService的实现又分为4个子实现，分别实现不同类型的功能模块
// RuntimeService interface should be implemented by a container runtime.
// The methods should be thread-safe.
type RuntimeService interface {
	RuntimeVersioner
	ContainerManager
	PodSandboxManager
	ContainerStatsManager

	// UpdateRuntimeConfig updates runtime configuration if specified
	UpdateRuntimeConfig(runtimeConfig *runtimeapi.RuntimeConfig) error
	// Status returns the status of the runtime.
	Status() (*runtimeapi.RuntimeStatus, error)
}

// 1. RuntimeVersioner contains methods for runtime name, version and API version.
type RuntimeVersioner interface {
	// Version returns the runtime name, runtime version and runtime API version
	Version(apiVersion string) (*runtimeapi.VersionResponse, error)
}

// 2. ContainerManager contains methods to manipulate containers managed by a
// container runtime. The methods are thread-safe.
type ContainerManager interface {
	// CreateContainer creates a new container in specified PodSandbox.
	CreateContainer(podSandboxID string, config *runtimeapi.ContainerConfig, sandboxConfig *runtimeapi.PodSandboxConfig) (string, error)
	// StartContainer starts the container.
	StartContainer(containerID string) error
	// StopContainer stops a running container with a grace period (i.e., timeout).
	StopContainer(containerID string, timeout int64) error
	// RemoveContainer removes the container.
	RemoveContainer(containerID string) error
	// ListContainers lists all containers by filters.
	ListContainers(filter *runtimeapi.ContainerFilter) ([]*runtimeapi.Container, error)
	// ContainerStatus returns the status of the container.
	ContainerStatus(containerID string) (*runtimeapi.ContainerStatus, error)
	// UpdateContainerResources updates the cgroup resources for the container.
	UpdateContainerResources(containerID string, resources *runtimeapi.LinuxContainerResources) error
	// ExecSync executes a command in the container, and returns the stdout output.
	// If command exits with a non-zero exit code, an error is returned.
	ExecSync(containerID string, cmd []string, timeout time.Duration) (stdout []byte, stderr []byte, err error)
	// Exec prepares a streaming endpoint to execute a command in the container, and returns the address.
	Exec(*runtimeapi.ExecRequest) (*runtimeapi.ExecResponse, error)
	// Attach prepares a streaming endpoint to attach to a running container, and returns the address.
	Attach(req *runtimeapi.AttachRequest) (*runtimeapi.AttachResponse, error)
	// ReopenContainerLog asks runtime to reopen the stdout/stderr log file
	// for the container. If it returns error, new container log file MUST NOT
	// be created.
	ReopenContainerLog(ContainerID string) error
}

// 3. PodSandboxManager contains methods for operating on PodSandboxes. The methods
// are thread-safe.
type PodSandboxManager interface {
	// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
	// the sandbox is in ready state.
	RunPodSandbox(config *runtimeapi.PodSandboxConfig, runtimeHandler string) (string, error)
	// StopPodSandbox stops the sandbox. If there are any running containers in the
	// sandbox, they should be force terminated.
	StopPodSandbox(podSandboxID string) error
	// RemovePodSandbox removes the sandbox. If there are running containers in the
	// sandbox, they should be forcibly removed.
	RemovePodSandbox(podSandboxID string) error
	// PodSandboxStatus returns the Status of the PodSandbox.
	PodSandboxStatus(podSandboxID string) (*runtimeapi.PodSandboxStatus, error)
	// ListPodSandbox returns a list of Sandbox.
	ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error)
	// PortForward prepares a streaming endpoint to forward ports from a PodSandbox, and returns the address.
	PortForward(*runtimeapi.PortForwardRequest) (*runtimeapi.PortForwardResponse, error)
}

// 4. ContainerStatsManager contains methods for retrieving the container
// statistics.
type ContainerStatsManager interface {
	// ContainerStats returns stats of the container. If the container does not
	// exist, the call returns an error.
	ContainerStats(containerID string) (*runtimeapi.ContainerStats, error)
	// ListContainerStats returns stats of all running containers.
	ListContainerStats(filter *runtimeapi.ContainerStatsFilter) ([]*runtimeapi.ContainerStats, error)
}


```



#### CNI接口源码

```go

type CNI interface {
   AddNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
   CheckNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
   DelNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
   GetNetworkListCachedResult(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
   GetNetworkListCachedConfig(net *NetworkConfigList, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

   AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
   CheckNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
   DelNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
   GetNetworkCachedResult(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
   GetNetworkCachedConfig(net *NetworkConfig, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

   ValidateNetworkList(ctx context.Context, net *NetworkConfigList) ([]string, error)
   ValidateNetwork(ctx context.Context, net *NetworkConfig) ([]string, error)
}
```





## k8s的可扩展性



![kubernetes-extensions](https://img.draveness.me/kubernetes-extensions-2021-03-10-16153845432607.png)



k8s通过定义接口的方式，解耦了许多底层组件，极大的增强了k8s的可扩展性。其中网络和容器的通过接口实现可拓展性的方式在上文讲述容器运行时的部分有更为详细的阐述。

## k8s的局限性

虽然k8s是云原生时代事实上的容器编排标准，但并不代表其完美无缺，这里对k8s现存的一些问题和一些解决方案的探索做一个简要的阐述。

### 集群管理

#### 水平伸缩性

首先是几圈

Kubernetes 社区对外宣传的是单个集群最多支持 5,000 节点，Pod 总数不超过 150,000，容器总数不超过 300,000 以及单节点 Pod 数量不超过 100 个，与几万节点的 Apache Mesos 集群、50,000 节点的微软 YARN 集群相比，Kubernetes 的集群规模整整差了一个数量级。虽然阿里云的工程师也通过优化 Kubernetes 的各个组件实现了 5 位数的集群规模，但是与其他的资源管理方式相比却有比较大的差距。

### 多集群管理

k8s的多集群管理会带来 资源不平衡、跨集群访问困难等问题

#### 现有的一些探索方案

##### kubefed

kubefed(Kubernetes Cluster Federation)是k8s社区给出的解决方案，提供了跨集群资源和网络管理的功能

![concepts.png](https://gitee.com/weixiao619/pic/raw/master/concepts.png)



### 应用场景

#### 应用分发

Kubernetes 主项目提供了几种部署应用的最基本方式，分别是 `Deployment`、`StatefulSet` 和 `DaemonSet`，这些资源分别适用于无状态服务、有状态服务和节点上的守护进程，这些资源能够提供最基本的策略，但是它们无法处理更加复杂的应用

很多常见的场景，例如只运行一次的 DaemonSe以及金丝雀和蓝绿部署等功能，现在的资源也存在很多问题，例如 StatefulSet 在初始化容器中卡住无法回滚和更新。

#### 批处理调度

机器学习、批处理任务和流式任务等工作负载的运行从 Kubernetes 诞生第一天起到今天都不是它的强项，大多数的公司都会使用 Kubernetes 运行在线服务处理用户请求，用 Yarn 管理的集群运行批处理的负载。当然在批处理任务方面，k8s社区本身也做了部分改进，有kube-batch的插件主要用于优化批处理任务的调度，同时华为在kube-batch的基础上演进出Volcano调度器有效的提高了k8s集群中运行批处理作业的能力，获得了广泛的认可。在后续比较云原生调度器的内容中会进一步对Volcano调度器做更为详细的阐述。

#### 硬多租户

今天的 Kubernetes 还很难做到硬多租户支持，也就是同一个集群的多个租户不会相互影响，也感知不到彼此的存在。尽管目前k8s用命名空间来划分虚拟集群，但也很难实现很好的资源隔离。

## Volcano 调度器

根据上一节的内容可以看出k8s原生的调度器，在调度的最后阶段（打分）是串行的调度每个pod，但是在AI训练或者大数据处理的场景下，多个实例需要进行相互协作，此时，实际的需求为需要保证一组实例同时运行成功，而k8s原生调度器无法满足此需求，因此kube-batch分组调度器应运而生，Volcano调度器则基于kube-batch进行了进一步的优化。

![img](https://gitee.com/weixiao619/pic/raw/master/scheduler-20210605222117986.PNG)

如图为Volcano调度器的工作流，一个job会打包多个pod形成一个PodGroup。

### 调度工作流

Volcano scheduler的工作流程如下：

1. 客户端提交的Job被scheduler观察到并缓存起来。
2. 周期性的开启会话，一个调度周期开始。
3. 将没有被调度的Job发送到会话的待调度队列中。
4. 遍历所有的待调度Job，按照定义的次序依次执行enqueue、allocate、preempt、reclaim、backfill等动作，为每个Job找到一个最合适的节点。将该Job 绑定到这个节点。action中执行的具体算法逻辑取决于注册的plugin中各函数的实现。
5. 关闭本次会话。

#### Action作用

##### enqueue

Enqueue action负责通过一系列的过滤算法筛选出符合要求的待调度任务并将它们送入待调度队列。经过这个action，任务的状态将由pending变为inqueue。

##### allocate

Allocate action负责通过一系列的预选和优选算法筛选出最适合的节点。

##### preempt

Preempt action负责根据优先级规则为同一队列中高优先级任务执行抢占调度。

##### reclaim

Reclaim action负责当一个新的任务进入待调度队列，但集群资源已不能满足该任务所在队列的要求时，根据队列权重回收队列应得资源。

##### backfill

backfill action负责将处于pending状态的任务尽可能的调度下去以保证节点资源的最大化利用。

### 一些调度插件

#### Gang Scheduling

以组为单位进行调度，

![image.png](https://gitee.com/weixiao619/pic/raw/master/1568282331579096.png)

#### DRF (dominant resource fairss)

优先调度请求资源较少的实例，yarn和mesos均有对应调度Sunday

​	![image.png](https://gitee.com/weixiao619/pic/raw/master/1568282358178951.png)



#### binpack

该调度算法尽量先填满已有节点，将负载聚拢到部分节点，有利于弹性伸缩，减少资源碎片

![image.png](https://gitee.com/weixiao619/pic/raw/master/1568282383798065.png)

#### proportion(队列)

与yarn的capacity Scheduler调度器类似，根据组织分配资源比例，在组织内部使用FIFO的队列模式进行资源调度

![image.png](https://gitee.com/weixiao619/pic/raw/master/1568282412649140-20210605213707655.png)

#### 最终优选

Volcano支持各种调度算法的插件，不同插件的调度逻辑可能出现相互冲突，因此Volcano可以给不同的调度插件设置不同的权重，根据加权值来最终决定po d的调度。

## 调度算法的演进

![image-20210605221823380](https://gitee.com/weixiao619/pic/raw/master/image-20210605221823380.png)

1. 单阶段

   使用一个单一的中心化调度器，调度所有资源，同时没有并发，但是集群规模过大时无法很好的处理。同时新增调度的规则变得十分困难，因为所有的用户资源使用同样的调度逻辑。

2. 两阶段调度

   增强了集群的弹性，但是受到资源可见性和锁的双重限制，导致很难调度挑剔的节点

3. 优化的两阶段调度

   使用共享状态，进行乐观的无锁并发控制。Omega系统使用了该架构，增强了延展性和性能。



### 问题分类

对调度问题进行分类

资源选择

##### 互相干扰

对资源进行调度时，如何解决并发冲突。

1.加锁

2.回撤

#### 分配粒度



#### 集群行为



![image-20210605115450361](https://gitee.com/weixiao619/pic/raw/master/image-20210605115450361.png)



#### 集中调度

通过中央式调度器有且仅有一个实例，没有并行，而且在一个单一的代码库，必须实现所有的策略选择。这种方法常见于高性能计算（HPC），对于多种策略的支持有两种方式：

1.通过复杂的计算，针对多个权重因子计算出整体的优先级

2.调度逻辑通过调用不同的代码逻辑进行分离。谷歌之前采用的就是这种，虽然经过各种优化使得它有较好的性能和拓展性表现，但是最后决定将调度算法改进为支持并发、独立调度组件的首要原因是出于软件工程角度的考虑。

#### 静态分区调度

将资源分为不同的区，每个不同的区域采用不同的调度方式，区域之间互相隔离

#### 两层调度器

解决静态分区调度器的一个显而易见的方案是实现分区的动态化，这就是两层调度器的工作原理，通过一个中心来对各个分区的资源进行协调分配，这种两层调度被Mesos和Hadoop-on-Demand广泛采用

mesos基于DRF策略和offer-based（主动提供）的方式由master向各个计算框架分配资源，并且提供了过滤器机制，计算框架可以事先描述自己需要的资源特征。该架构适用于短任务，并且存在一定的缺点：（1）offer-based的方式是悲观锁实现的，每次只能给一个计算框架分配资源，导致并发度低；（2）每次的分配都是基于当前状况进行的，计算框架并不能感知到系统的资源情况，所以分配不是全局最优的；（3）逐个计算框架进行分配导致系统不能够支持资源抢占；（4）当系统中被短任务填充时，会导致长任务饿死；（5）资源保持可能会导致死锁现象的发生；



Yarn采用的是可能是一种两层调度，但更多的是集中调度：

yarn是基于资源申请的方式进行资源分配的。AM向RM请求资源，RM基于NM的心跳触发分配，按照一定的策略将资源分配给AM，AM与NM通信运行任务。目前的yarn还存在一些缺点：目前的AM只负责进行任务管理并没有提供调度的能力，这也是一部分人将YARN划分为集中式调度器的原因；目前仅支持对内存的分配；虽然现在yarn支持AM在进行资源申请的时候选择放置偏好点，但是并没有说明这些偏好是根据什么标准得到的。

#### 共享状态调度

针对两层调度系统存在的并行性和不能进行全局最优的调度问题，共享状态系统进行了优化。主要有两点措施，第一存在多个调度器，每个调度器都拥有系统全量的资源信息，可以根据该信息进行调度。第二调度器将其调度的结果以原子的方式提交给cell state维护模块，由其决定本次提交是否成功，这里体现了乐观并发的思想。

每个调度器会定期以私有的方式更新自己副本中的cell state；
每个调度器可以设置其私有得调度策略，有很大的自由度；
Omega只是将优先级这一限制放到了共享数据的验证代码中，即当同时由多个应用程序申请同一份资源时，优先级最高的那个应用程序将获得该资源，其他资源限制全部下放到各个子调度器。
对整个集群中的所有资源分组，限制每类应用程序的资源使用量，限制每个用户的资源使用量等，这些全部由各个应用程序调度器自我管理和控制。
引入多版本并发控制后，限制该机制性能的一个因素是资源访问冲突的次数，冲突次数越多，系统性能下降的越快，而google通过实际负载测试证明，这种方式的冲突次数是完全可以接受的。



### Borg

![image-20210605222630812](https://gitee.com/weixiao619/pic/raw/master/image-20210605222630812.png)



![image-20210605222841624](https://gitee.com/weixiao619/pic/raw/master/image-20210605222841624.png)

如图所示为Borg的架构

### Yarn

Yarn有三种调度器可供选择，分别为：

#### FIFO Scheduler 

应用按照提交的顺序排成队列，先进先出

#### capacity Scheduler (默认调度器)

容量调度器允许多个组织共享整个集群，在组织之间划分资源配额，但是在组织内部使用FIFO的模式进行调度

#### Fair Scheduler(CDH版本Hadop采用)

公平调度器，为所有应用以一种公平的方式分配资源，通过参数的设置，可以对公平有不同的定义。

公平调度器 Fair Scheduler 最初是由 Facebook 开发设计使得 Hadoop 应用能够被多用户公平地共享整个集群资源，现被 Cloudera CDH 所采用。Fair Scheduler 不需要保留集群的资源，因为它会动态在所有正在运行的作业之间平衡资源。



## 参考文献

1. [谈谈k8s的问题和局限性](https://draveness.me/kuberentes-limitations/)
2. [为什么k8s要替换docker](https://draveness.me/whys-the-design-kubernetes-deprecate-docker/)
3. https://en.wikipedia.org/wiki/Linux_namespaces
4. [cgroups简介](https://tech.meituan.com/2015/03/31/cgroups.html)





