# Kubernetes

# 1 Kubernetes 是什么

尽管 Docker 为容器化的应用程序提供了开放标准，但随着容器越来越多，出现了一系列新问题：

- 单机不足以支持更多的容器
- 分布式环境下容器如何通信
- 如何协调和调度这些容器
- 如何在升级应用程序时不中断服务
- 如何监视应用程序的运行状况
- 如何批量重新启动容器里的程序
- ……

Google 利用在容器管理方面多年的经验和专业知识，在 2014 年推出了 Kubernetes。Kubernetes 的设计源自 Google 内部一个久负盛名的大规模集群管理系统——**Borg**，可以看作 Borg 设计思想的开源实现，其目的是实现资源管理的自动化以及跨数据中心的资源利用率最大化，凝聚了 Google 十几年来大规模应用容器技术的经验积累。

![](imgs/1.png)

如今，Kubernetes 已成为容器编排领域的事实标准，由 CNCF（Cloud Native Computing Foundation，云原生计算基金会）托管，开源并公布了一系列标准化方法，主流的公有云平台都提供了托管的 Kubernetes 服务。

**Kubernetes** 源于希腊语，它的中文翻译是“舵手”或者“领航员”。在一些常见的资料中也会看到“K8s”这个词，它是通过将中间的 8 个字母“ubernete”替换为“8”而得到的缩写。

在学习 Docker 的文章中介绍过一个概念叫作 Container，Container 这个英文单词除了容器还有另外一个意思，就是“集装箱”。Kubernetes 借用了这一寓意，把自己定位为运送和管理集装箱（即容器）的货轮。

更具体地说：**Kubernetes 是一个自动化的容器编排平台，它负责应用的部署、应用的弹性以及应用的管理，这些都是基于容器的**。

# 2 Kubernetes 有什么用

Kubernetes 有如下几个核心的功能：

- 服务的发现与负载的均衡
- 容器的自动装箱，我们也会把它叫作 **scheduling**，就是“调度”，把一个容器放到一个集群的某一个机器上
- Kubernetes 会帮助我们去做存储的编排，让存储的生命周期与容器的生命周期能有一个连接
- Kubernetes 会帮助我们去做自动化的容器的恢复。在一个集群中，经常会出现宿主机的问题或者说是 OS 的问题，导致容器本身的不可用，Kubernetes 会自动地对这些不可用的容器进行恢复
- Kubernetes 会帮助我们去做应用的自动发布与应用的回滚，以及与应用相关的配置（ConfigMap）与敏感信息（Secret）的管理
- 对于 Job 类型任务，Kubernetes 可以去做批量的执行
- 为了让这个集群、这个应用更富有弹性，Kubernetes 也支持水平的伸缩

下面，我们希望通过三个例子更切实地介绍一下 Kubernetes 的能力。

## 2.1 调度

Kubernetes 可以把用户提交的容器放到 Kubernetes 管理的集群的某一台节点上去。Kubernetes 的调度器是执行这项能力的组件，它会观察正在被调度的这个容器的大小、规格。

比如说它所需要的 CPU 以及它所需要的内存，然后在集群中找一台相对比较空闲的机器，把容器放置（Placement）上去。在这个例子中，它可能会把红颜色的这个容器放置到第二个空闲的机器上，来完成一次调度的工作。

![](imgs/2.png)

## 2.2 自动修复

Kubernetes 有一个节点健康检查的功能，它会监测这个集群中所有的宿主机，当宿主机本身出现故障，或者软件出现故障的时候，这个节点健康检查会自动对它进行发现。接下来 Kubernetes 会把运行在这些失败节点上的容器进行自动迁移，迁移到一个正在健康运行的宿主机上，来完成集群内容器的一个自动恢复。

![](imgs/3.png)

![](imgs/4.png)

## 2.3 水平伸缩

Kubernetes 有业务负载检查的能力，它会监测业务上所承担的负载，如果这个业务本身的 CPU 利用率过高，或者响应时间过长，它可以对这个业务进行一次扩容。

比如说在下面的例子中，黄颜色的业务过度忙碌，Kubernetes 就可以把黄颜色负载从一份变为三份。接下来，它就可以通过负载均衡把原来打到第一个黄颜色上的负载平均分到三个黄颜色的负载上去，以此来缩短响应时间。

![](imgs/5.png)

这项能力由 HPA（Horizontal Pod Autoscaler，Pod 水平自动伸缩）提供，它默认根据 CPU、内存利用率或自定义指标调整副本数。对于消息消费这类事件驱动的业务，还可以引入 CNCF 毕业项目 KEDA（Kubernetes Event-driven Autoscaling），根据 Kafka 消息积压量、Prometheus 中的业务指标等外部事件来触发扩缩容，并且支持在没有负载时把副本数缩减到零，有消息到达时再拉起，这是单独使用 HPA 做不到的。

# 3 架构

![](imgs/6.png)

Kubernetes 架构是一个比较典型的 server-client 架构。

控制平面（Control Plane）作为集群的中央管控枢纽，会去与 Node 进行连接。所有 UI、客户端这些用户侧的组件，只会和控制平面进行连接，把希望的状态或者想执行的命令提交给控制平面，再由控制平面把这些命令或者状态下发给相应的节点，进行最终的执行。

## 3.1 控制平面

![](imgs/7.png)

控制平面在早期版本中被称为 Master（图中仍沿用这一叫法），官方文档现已统一使用 Control Plane，它包含四个主要的组件：

- **API Server**：整个集群的控制中枢，提供集群中各个模块之间的数据交换，并将集群状态和信息存储到分布式键值（key-value）存储系统 etcd 集群中。同时它也是集群管理、资源配额、提供完备的集群安全机制的入口，为集群各类资源对象提供增删改查以及 watch 的 REST API
- **Controller**：集群状态管理器（即 Controller Manager），以保证 Pod 或其他资源达到期望值。当集群中某个 Pod 的副本数或其他资源因故障和错误导致无法正常运行，没有达到设定的值时，Controller Manager 会尝试自动修复并使其达到期望状态
- **Scheduler**：集群 Pod 的调度中心，主要是通过调度算法将 Pod 分配到最佳的节点（Node）。它通过 API Server 监听所有 Pod 的状态，一旦发现新的未被调度到任何 Node 的 Pod（`spec.nodeName` 为空），就会根据一系列策略选择最佳节点进行调度，对每一个 Pod 创建一个绑定（binding），然后被调度节点上的 kubelet 负责启动该 Pod
- **etcd**：最初由 CoreOS 开发（CoreOS 于 2018 年被 Red Hat 收购，同年 etcd 被捐给 CNCF，2020 年毕业），用于可靠地存储集群的配置数据，是一种持久性、轻量型、分布式的键值（key-value）数据存储组件。etcd 作为 Kubernetes 集群的持久化存储系统，集群的灾难恢复和状态信息存储都与其密不可分，所以在 Kubernetes 高可用集群中，etcd 的高可用是至关重要的

我们刚刚提到的 API Server，它本身在部署结构上是一个可以水平扩展的组件；Controller 是一个可以进行热备的组件，它只有一个 active 实例；调度器也是类似的，虽然只有一个 active 实例，但是可以进行热备。

## 3.2 Node

Kubernetes 的 Node 是真正运行业务负载的，每个业务负载会以 **Pod** 的形式运行，一个 Pod 中运行一个或者多个容器。Node 上的主要组件如下：

![](imgs/8.png)

- **kubelet**：控制平面在 Node 上的 Agent，也是 Node 上最为关键的组件。它通过 API Server 获取本节点需要运行的 Pod，将每个 Pod 转换成一组容器交给 Container Runtime 运行，并负责维护容器的生命周期，比如创建容器、为 Pod 挂载数据卷、下载 Secret、上报容器和节点状态等
- **kube-proxy**：负责为 Service 提供 cluster 内部的服务发现和负载均衡，在 Node 上实现 Pod 网络代理，维护网络规则和四层负载均衡工作
- **Container Runtime**：负责镜像管理以及 Pod 和容器的真正运行。常见的容器运行时有 containerd、CRI-O 等，它们通过 CRI（Container Runtime Interface）与 kubelet 交互。Kubernetes 在 1.24 版本中移除了 dockershim，不再直接对接 Docker Engine，但用 Docker 构建的镜像符合 OCI 标准，依然可以正常运行
- **Storage Plugin** 与 **Network Plugin**：在 OS 上去创建容器所需要运行的环境，最终把容器或者 Pod 运行起来，也需要对存储跟网络进行管理。Kubernetes 并不会直接进行网络存储的操作，而是通过标准接口交给插件完成：存储走 CSI（Container Storage Interface），网络走 CNI（Container Network Interface），存储厂商、云厂商或社区会提供相应的 CSI 驱动和 CNI 插件

在网络这一侧，近几年基于 eBPF 的方案发展很快。以 Cilium 为代表的 CNI 插件可以直接在内核中完成 Service 转发，从而完全替代 kube-proxy，在大规模集群中可以获得更好的性能和可观测性。kube-proxy 自身也在演进：nftables 模式在 1.33 版本 GA，IPVS 模式则在 1.35 版本被标记为弃用。

下面我们以一个例子再去看一下 Kubernetes 架构中的这些组件是如何互相进行交互的。

![](imgs/9.png)

用户可以通过 UI 或者 CLI 提交一个 Pod 给 Kubernetes 进行部署，这个 Pod 请求首先会通过 CLI 或者 UI 提交给 Kubernetes API Server，下一步 API Server 会把这个信息写入到它的存储系统 etcd，之后 Scheduler 会通过 API Server 的 watch 或者叫作 notification 机制得到这个信息：有一个 Pod 需要被调度。

这个时候 Scheduler 会根据它缓存的集群状态进行一次调度决策，选出合适的节点后，通过 API Server 为这个 Pod 创建一个绑定（Binding），把 Pod 指派到该节点上。

API Server 接收到这次操作之后，会把这次的结果再次写到 etcd 中。相应节点的 kubelet 一直通过 watch 机制关注着分配给本节点的 Pod，它发现这个新 Pod 后，就会调用 Container Runtime 来真正启动这个容器并配置它的运行环境，通过 CSI 驱动配置存储，通过 CNI 插件配置网络。

# 4 核心概念

## 4.1 Pod

**Pod 是 Kubernetes 的一个最小调度以及资源单元**。

Pod 可以理解为一个“容器组”，其中装的是由容器运行时（如 containerd，早期多为 Docker）创建的容器。Pod 相当于独立主机，可以封装一个或者多个容器。Pod 有自己的 IP 地址、主机名，相当于一台独立的沙箱环境。处于同一个 Pod 中的容器共享同样的存储空间（Volume，卷或存储卷）、网络命名空间、IP 地址和端口（Port）。

比如像下面的这幅图里面，它包含了两个容器，每个容器可以指定它所需要的资源大小。比如说，1 个核、1 GB 内存，或者说 0.5 个核、0.5 GB 内存。当然在这个 Pod 中也可以包含一些其他所需要的资源：比如说我们所看到的 Volume 卷这个存储资源，比如说我们需要 100 GB 的存储或者 20 GB 的另外一个存储。

![](imgs/10.png)

在 Pod 里面，我们也可以去定义容器所需要运行的方式。比如说运行容器的 Command，以及运行容器的环境变量等等。Pod 这个抽象也给这些容器提供了一个共享的运行环境，它们会共享同一个网络环境，这些容器可以用 localhost 来进行直接的连接。而 Pod 与 Pod 之间，是互相有 isolation 隔离的。

## 4.2 Deployment

Deployment 是在 **Pod 这个抽象上更为上层的一个抽象**，它可以定义一组 Pod 的副本数目以及这个 Pod 的版本。一般大家用 Deployment 这个抽象来做应用的真正的管理，而 Pod 是组成 Deployment 最小的单元。

Kubernetes 是通过 Controller，也就是我们刚才提到的控制器去维护 Deployment 中 Pod 的数目，它也会去帮助 Deployment 自动恢复失败的 Pod。

比如说我可以定义一个 Deployment，这个 Deployment 里面需要两个 Pod，当一个 Pod 失败的时候，控制器就会监测到，它会通过新生成一个 Pod，把 Deployment 中的 Pod 数目从一个恢复到两个。通过控制器，我们也可以完成发布的策略，比如说进行滚动升级（RollingUpdate）、重建（Recreate）升级，或者进行版本的回滚。

![](imgs/11.png)

Deployment 并不直接管理 Pod，而是通过 **ReplicaSet**（RS，属于 `apps/v1`）来维持 Pod 的副本数量，每次发布都会生成一个新的 ReplicaSet，回滚就是切回旧的 ReplicaSet。早期承担这一职责的是 **Replication Controller**（RC），它属于 core `v1` 中的 Legacy API，只支持基于等式的 Label Selector，而 ReplicaSet 还支持基于集合的 Label Selector。如今官方推荐使用 Deployment（配合 ReplicaSet）来管理副本，RC 已基本不再使用。

## 4.3 Volume

Volume 就是**卷**的概念，它是用来管理 Kubernetes 存储的，是用来声明在 Pod 中的容器可以访问文件目录的，一个卷可以被挂载在 Pod 中一个或者多个容器的指定路径下面。

而 Volume 本身是一个抽象的概念，一个 Volume 可以去支持多种后端存储。比如说本地存储，Ceph 这样的分布式存储，阿里云、AWS、Google Cloud 上的云盘等。如今这些后端存储基本都通过 CSI 驱动接入，早期内置在 Kubernetes 代码中的各类存储插件（in-tree 插件）已陆续迁移到 CSI 或被移除。

![](imgs/12.png)

在使用虚拟机的情况下，我们通常会先定义一个网络存储，然后从中划出一个“网盘”并挂接到虚拟机上。**Persistent Volume**（PV）和与之相关联的 **Persistent Volume Claim**（PVC）也起到了类似的作用。PV 可以被理解成 Kubernetes 集群中的某个网络存储对应的一块存储，它与 Volume 类似，但有以下区别：

- PV 一般是网络存储，不属于任何 Node，但可以在每个 Node 上访问（此外也支持 local 类型的本地持久卷）
- PV 并不是被定义在 Pod 上的，而是独立于 Pod 之外定义的

## 4.4 Service

**Service 提供了一个或者多个 Pod 实例的稳定访问地址**。

Service 在 Kubernetes 中定义了一个服务的访问入口地址，前端的应用（Pod）通过这个入口地址访问其背后的一组由 Pod 副本组成的集群实例，Service 与其后端 Pod 副本集群之间则是通过 Label Selector 来实现无缝对接的。

![](imgs/13.png)

一个 Deployment 可能有两个甚至更多个完全相同的 Pod。对于一个外部的用户来讲，访问哪个 Pod 其实都是一样的，所以它希望做一次负载均衡，在做负载均衡的同时，用户只想访问某一个固定的 VIP，也就是 **Virtual IP** 地址，而不希望得知每一个具体的 Pod 的 IP 地址。

对一个外部用户来讲，如果提供多个具体的 Pod 地址，这个用户就要不停地去更新 Pod 地址，尤其是当 Pod 失败重启之后。因此我们希望有一个抽象，把所有 Pod 的访问能力抽象成一个第三方的 IP 地址，实现这个的 Kubernetes 抽象就叫 Service。

**Kubernetes 通过给 Service 分配静态 IP 地址和域名来提供服务发现机制**，并将流量负载均衡到与选择器匹配的 Pod 上（即使是故障导致 Pod 从一台机器移动到另一台机器）。具体的转发由 kube-proxy 完成：Linux 上默认的 iptables 模式会随机选择一个后端 Pod；IPVS 模式支持轮询（rr）、最少连接（lc）、源地址哈希（sh）等调度算法，但已在 1.35 版本被标记为弃用；nftables 模式已经 GA，未来将成为新的默认模式。

实现 Service 有多种方式，Kubernetes 支持 Cluster IP，也就是上面我们讲过的 kube-proxy 的组网，它也支持 NodePort、LoadBalancer 等其他的一些访问能力。

![](imgs/14.png)

其主要类型有：

- **ClusterIP**：虚拟的服务 IP 地址，该地址用于 Kubernetes 集群内部的 Pod 访问，在 Node 上 kube-proxy 通过设置的 iptables（或 nftables、IPVS）规则进行转发
- **NodePort**：使用宿主机的端口，使能够访问各 Node 的外部客户端通过 Node 的 IP 地址和端口号就能访问服务
- **LoadBalancer**：使用外接负载均衡器完成到服务的负载分发，外部负载均衡器的 IP 地址通常由云服务商分配并写入 Service 的 `status.loadBalancer` 字段，通常用于公有云

Service 工作在四层，七层的 HTTP 路由过去主要依靠 Ingress。如今 Ingress API 已被冻结、不再新增功能，社区推荐使用 2023 年 10 月 GA 的 **Gateway API**，它的表达能力更强，也更适合多团队分工；广泛使用的 ingress-nginx 控制器也已于 2026 年 3 月退役。在服务网格领域，Istio 的 Ambient 模式在 1.24 版本 GA，它不再需要为每个 Pod 注入 Sidecar，资源开销明显降低。

## 4.5 Label

Label 是 Kubernetes 系统中的一个核心概念，**一个 Label 表示一个 key=value 的键值对**，key、value 的值由用户指定。Label 可以被附加到各种资源对象上，例如 Node、Pod、Service、ReplicaSet 等，一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上。Label 通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。给一个资源对象定义了 Label 后，我们随后可以通过 **Label Selector** 查询和筛选拥有这个 Label 的资源对象，来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

Label Selector 当前有两种表达式，基于等式的和基于集合的：

- `name=redis-slave`：匹配所有具有标签 `name=redis-slave` 的资源对象
- `env!=production`：匹配所有不具有标签 `env=production` 的资源对象
- `name in(redis-master, redis-slave)`：匹配所有具有标签 `name=redis-master` 或者 `name=redis-slave` 的资源对象
- `name not in(php-frontend)`：匹配所有不具有标签 `name=php-frontend` 的资源对象

以 my-web Pod 为例（资源名称须符合 DNS 命名规范，只能包含小写字母、数字和 `-`）：

```yaml
apiVersion: v1  # 分组和版本
kind: Pod       # 资源类型
metadata:
  name: my-web  # Pod名
  labels:
    app: my-web # Pod的标签
```

当一个 Service 的 selector 中指明了这个 Pod 时，该 Pod 就会与该 Service 绑定：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web
spec:
  selector:
    app: my-web
  ports:
  - port: 8080
```

## 4.6 StatefulSet

在 Kubernetes 系统中，Pod 的管理对象 ReplicaSet、Deployment、DaemonSet 和 Job 都面向无状态的服务。但现实中有很多服务是有状态的，特别是一些复杂的中间件集群，例如 MySQL 集群、MongoDB 集群、Kafka 集群、ZooKeeper 集群等，这些应用集群有 4 个共同点：

- 每个节点都有固定的身份 ID，通过这个 ID，集群中的成员可以相互发现并通信
- 集群的规模是比较固定的，集群规模不能随意变动
- 集群中的每个节点都是有状态的，通常会持久化数据到永久存储中
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损

StatefulSet 保证 Pod 重新建立后，Hostname 不会发生变化，Pod 就可以通过 Hostname 来关联数据。因此，StatefulSet 具有以下特点：

- StatefulSet 里的每个 Pod 都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设 StatefulSet 的名称为 kafka，那么第 1 个 Pod 叫 kafka-0，第 2 个叫 kafka-1，以此类推
- StatefulSet 控制的 Pod 副本的启停顺序是受控的：扩容时按 0 到 N-1 的顺序逐个创建，前一个 Pod 处于 Running 且 Ready 状态后才会创建下一个；缩容时则按相反顺序逐个删除。如果应用不需要这种顺序保证，可以把 `podManagementPolicy` 设为 `Parallel` 来并行启停
- StatefulSet 里的 Pod 采用稳定的持久化存储卷，通过 PV 或 PVC 来实现，删除 Pod 时默认不会删除与 StatefulSet 相关的存储卷（为了保证数据的安全）
- StatefulSet 除了要与 PV 卷捆绑使用以存储 Pod 的状态数据，还要与 Headless Service 配合使用

**Headless Service** 与普通 Service 的关键区别在于，它没有 Cluster IP，如果解析 Headless Service 的 DNS 域名，则返回的是该 Service 对应的全部 Pod 的 Endpoint 列表。

不过 StatefulSet 只解决了身份和存储的问题，数据库主从切换、备份恢复、版本升级这类运维逻辑它并不关心。为此社区发展出了 **Operator** 模式：通过 CRD（Custom Resource Definition）定义一种新的资源，再编写对应的控制器，把领域运维知识编码进去，让 Kubernetes 像管理 Deployment 一样管理一个数据库集群。Prometheus Operator、CloudNativePG 等都是这种模式的典型代表，如今在 Kubernetes 上运行有状态中间件，通常会优先选择成熟的 Operator。

## 4.7 Namespace

Namespace 是用来做一个集群内部的逻辑隔离的，它包括鉴权、资源管理等。Kubernetes 的每个资源，比如刚才讲的 Pod、Deployment、Service 都属于一个 Namespace，同一个 Namespace 中的资源需要保证命名的唯一性，不同的 Namespace 中的资源可以重名。

Namespace 的一个用例：比如在阿里巴巴，内部会有很多个业务单元（Business Unit，BU），每个 BU 之间希望有一个视图上的隔离，并且在鉴权上不一样，在配额（quota）上也不一样，我们就会用 Namespace 来给每个 BU 提供一个视图上的隔离机制。

![](imgs/15.png)
