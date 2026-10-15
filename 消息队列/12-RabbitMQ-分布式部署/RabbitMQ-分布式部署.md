RabbitMQ分布式部署有3种方式：

- 集群
- Federation
- Shovel

Federation与Shovel都是以插件的形式来实现，复杂性相对高，而集群是RabbitMQ的自带属性，相对简单。

这三种方式并不是互斥的，可以根据需求选择相互组合来达到目的。

# 1、集群

## 1.1 基本概念

RabbitMQ本身是基于Erlang编写，Erlang语言天生具备分布式特性（通过同步Erlang集群各节点的magic cookie来实现）。

因此，RabbitMQ天然支持Clustering。这使得RabbitMQ本身不需要像ActiveMQ、Kafka那样通过ZooKeeper分别来实现HA方案和保存集群的元数据。集群是保证可靠性的一种方式，同时可以通过水平扩展以达到增加消息吞吐量能力的目的。

我们把部署RabbitMQ的机器称为节点，也就是broker。broker有2种类型节点：**磁盘节点**和**内存节点**。顾名思义，磁盘节点的broker把元数据存储在磁盘中，内存节点把元数据存储在内存中，很明显，磁盘节点的broker在重启后元数据可以通过读取磁盘进行重建，保证了元数据不丢失，内存节点的broker可以获得更高的性能，但在重启后元数据就都丢了。

元数据包含以下内容：

- queue元数据：queue名称、属性
- exchange：exchange名称、属性
- binding元数据：exchange和queue之间、exchange和exchange之间的绑定关系
- vhost元数据：vhost内部的命名空间、安全属性数据等

**单节点系统必须是磁盘节点**，否则每次你重启RabbitMQ之后所有的系统配置信息都会丢失。

**集群中至少有一个磁盘节点**，当节点加入和离开集群时，必须通知磁盘	节点。

如果集群中的唯一一个磁盘节点，结果这个磁盘节点还崩溃了，那会发生什么情况？集群依然可以继续路由消息（因为其他节点元数据在还存在），但无法做以下操作：

- 创建队列、交换器、绑定
- 添加用户
- 更改权限
- 添加、删除集群节点

也就是说，如果唯一磁盘的磁盘节点崩溃，**集群是可以保持运行的，但不能更改任何东西**。为了增加可靠性，一般会在集群中设置两个磁盘节点，只要任何一个处于工作状态，就可以保障集群的正常服务。



RabbitMQ的集群模式分为两种：**普通模式**与**镜像模式**。

## 1.2 普通模式

普通模式，也是默认的集群模式。

对于Queue来说，**消息实体只存在于其中一个节点**，A、B两个节点仅有相同的元数据，即队列结构。当消息进入A节点的Queue中后，consumer从B节点拉取时，RabbitMQ会临时在A、B间进行消息传输，把A中的消息实体取出并经过B发送给consumer。所以consumer应尽量连接每一个节点，从中取消息。即对于同一个逻辑队列，要在多个节点建立物理Queue。否则无论consumer连A或B，出口总在A，会产生瓶颈。

![](1.png) 

队列所在的节点称为**宿主节点**。

队列创建时，只会在宿主节点创建队列的进程，宿主节点包含完整的队列信息，包括元数据、状态、内容等等。因此，**只有队列的宿主节点才能知道队列的所有信息**。

队列创建后，集群只会同步队列和交换器的元数据到集群中的其他节点，并不会同步队列本身，因此**非宿主节点就只知道队列的元数据和指向该队列宿主节点的指针**。 

假如现在一个客户端需要对Queue A进行发布或者订阅，发起与集群的连接，有两种可能的场景：

- 如果客户端连接至Broker A，Broker A是Queue A的宿主节点，那么此时的集群中的消息收发只与Broker A相关。
- 如果客户端连接至Broker B或Broker C，不是Queue A的宿主节点，那么此时的Broker主要起了一个路由转发作用，根据这两个节点上的元数据转发至Broker A上。

由于节点之间存在路由转发的情况，对延迟非常敏感，应当只在本地局域网内使用，在广域网中不应该使用集群，而应该用Federation或者Shovel代替。

这样的设计，保证了不论从哪个broker中均可以消费所有队列的数据，并分担了负载，因此，增加broker可以线性提高服务的性能和吞吐量。

但该方案也有显著的缺陷，那就是**不能保证消息不会丢失**。当集群中某一节点崩溃时，崩溃节点所在的队列进程和关联的绑定都会消失，附加在那些队列上的消费者也会丢失其订阅信息，匹配该队列的新消息也会丢失。比如A为宿主节点，当A节点故障后，B节点无法取到A节点中还未消费的消息实体。如果做了消息持久化，那么得等A节点恢复，然后才可被消费；如果没有持久化的话，然后就没有然后了……

肯定有不少同学会问，想要实现HA方案，那将RabbitMQ集群中的所有Queue的完整数据在所有节点上都保存一份不就可以了吗？比如类似MySQL的主主模式，任何一个节点出现故障或者宕机不可用时，那么使用者的客户端只要能连接至其他节点，不就能够照常完成消息的发布和订阅吗？ 

RabbitMQ这么设计是基于性能和存储空间上来考虑：

- 存储空间，如果每个集群节点都拥有所有Queue的完全数据拷贝，那么每个节点的存储空间会非常大，集群的消息积压能力会非常弱，无法通过集群节点的扩容提高消息积压能力。

- 性能，消息的发布者需要将消息复制到每一个集群节点，对于持久化消息，网络和磁盘同步复制的开销都会明显增，无法提升性能。（此处可以引申思考一下kafka中replica的分配方式） 。


## 1.3 镜像模式

引入**镜像队列**（Mirror Queue）的机制，可以将队列镜像到集群中的其他Broker节点之上，如果集群中的一个节点失效了，队列能够自动切换到镜像中的另一个节点上以保证服务的可用性。

一个镜像队列中包含有1个主节点master和若干个从节点slave。其主从节点包含如下几个特点：

- 消息的读写都是在master上进行，并不是读写分离

- master接收命令后会向salve进行组播，salve会命令执行顺序执行

- master失效，根据节点加入的时间，最老的slave会被提升为master

- 互为镜像的是队列，并非节点，集群中可以不同节点可以互为镜像队列，也就是说队列的master可以分布在不同的节点上

该模式和普通模式不同之处在于，消息实体会主动在镜像节点间同步，而不是在consumer取数据时临时拉取。该模式带来的副作用也很明显，除了降低系统性能外，如果镜像队列数量过多，加之大量的消息进入，集群内部的网络带宽将会被这种同步通讯大大消耗掉。所以在对可靠性要求较高的场合中适用。

### 1.3.1 镜像队列的设置

一个队列想做成镜像队列，需要先设置policy，然后客户端创建队列的时候，rabbitmq集群根据队列名称自动设置为普通队列还是镜像队列。

镜像队列的配置通过添加policy完成，policy添加的命令为：

> rabbitmqctl  set_policy  [-p Vhost]  Name  Pattern  Definition  [Priority]
>

- `-p Vhost`:  可选参数，针对指定vhost下的queue进行设置
- `Name`:  policy的名称
- `Pattern`:  queue的匹配模式（正则表达式）
- `Definition`:  镜像定义，包括三个部分 `ha-mode`，`ha-params`，`ha-sync-mode`
  -  `ha-mode`:  指明镜像队列的模式，有效值为 all/exactly/nodes
    - all表示在集群所有的节点上进行镜像
    - exactly表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
    - nodes表示在指定的节点上进行镜像，节点名称通过ha-params指定
  -  `ha-params`: ha-mode模式需要用到的参数
  - `ha-sync-mode`:  镜像队列中消息的同步方式，有效值为automatic，manually
- `Priority`:  可选参数， policy的优先级

例如，对队列名称以hello开头的所有队列进行镜像，并在集群的两个节点上完成镜像，policy的设置命令为：

`rabbitmqctl  set_policy  hello-ha  "^hello"  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'`

### 1.3.2 镜像队列的实现原理

#### 1.3.2.1 综述

通常队列由两部分组成：一部分是AMQQueue，负责AMQP协议相关的消息处理，即接收生产者发布的消息、向消费者投递消息、处理消息confirm、acknowledge等等；另一部分是BackingQueue，它提供了相关的接口供AMQQueue调用，完成消息的存储以及可能的持久化工作等。

镜像队列基本上就是一个特殊的BackingQueue，它内部包裹了一个普通的BackingQueue做本地消息持久化处理，在此基础上增加了将消息和ack复制到所有镜像的功能。所有对mirror_queue_master的操作，会通过组播GM（下面会讲到）的方式同步到各slave节点。GM负责消息的广播，mirror_queue_slave负责回调处理，而master上的回调处理是由coordinator负责完成。mirror_queue_slave中包含了普通的BackingQueue进行消息的存储，master节点中BackingQueue包含在mirror_queue_master中由AMQQueue进行调用。

消息的发布（除了Basic.Publish之外）与消费都是通过master节点完成。master节点对消息进行处理的同时将消息的处理动作通过GM广播给所有的slave节点，slave节点的GM收到消息后，通过回调交由mirror_queue_slave进行实际的处理。

![](3.jpeg)

**GM(Guarenteed Multicast)**是一种可靠的组播通讯协议，该协议能够保证组播消息的原子性，即保证组中活着的节点要么都收到消息要么都收不到。它的实现大致如下：

将所有的节点形成一个循环链表，每个节点都会监控位于自己左右两边的节点，当有节点新增时，相邻的节点保证当前广播的消息会复制到新的节点上；当有节点失效时，相邻的节点会接管保证本次广播的消息会复制到所有的节点。在master节点和slave节点上的这些gm形成一个group，group（gm_group）的信息会记录在mnesia中。不同的镜像队列形成不同的group。消息从master节点对于的gm发出后，顺着链表依次传送到所有的节点，由于所有节点组成一个循环链表，master节点对应的gm最终会收到自己发送的消息，这个时候master节点就知道消息已经复制到所有的slave节点了。

#### 1.3.2.2 新增节点

slave节点先从gm_group中获取对应group的所有成员信息，然后随机选择一个节点并向这个节点发送请求，这个节点收到请求后，更新gm_group对应的信息，同时通知左右节点更新邻居信息（调整对左右节点的监控）及当前正在广播的消息，然后回复通知请求节点成功加入group。请求加入group的节点收到回复后再更新rabbit_queue中的相关信息，并根据需要进行消息的同步。

![](4.jpg)

![](5.jpg)

#### 1.3.2.3 删除节点

当slave节点失效时，仅仅是相邻节点感知，然后重新调整邻居节点信息、更新rabbit_queue、gm_group的记录等。如果是master节点失效，"资格最老"的slave节点被提升为master节点，slave节点会创建出新的coordinator，并告知gm修改回调处理为coordinator，原来的mirror_queue_slave充当amqqueue_process处理生产者发布的消息，向消费者投递消息等。

![](6.jpg)



上面提到如果是slave节点失效，只有相邻的节点能感知到，那么master节点失效是不是也是只有相邻的节点能感知到？假如是这样的话，如果相邻的节点不是"资格最老"的节点，怎么通知"资格最老"的节点提升为新的master节点呢?

实际上，所有的slave节点在加入group时，mirror_queue_slave进程会对master节点的amqqueue_process进程（也可能是mirror_queue_slave进程）进行监控，如果master节点失效的话，mirror_queue_slave会感知，然后再通过gm进行广播，这样所有的节点最终都会知道master节点失效。当然，只有"资格最老"的节点会提升自己为新的master。

#### 1.3.2.4 消息的广播

消息从master节点发出，顺着节点链表发送。在这期间，所有的slave节点都会对消息进行缓存，当master节点收到自己发送的消息后，会再次广播ack消息，同样ack消息会顺着节点链表经过所有的slave节点，其作用是通知slave节点可以清除缓存的消息，当ack消息回到master节点时对应广播消息的生命周期结束。

下图为一个简单的示意图，A节点为master节点，广播一条内容为"test"的消息。"1"表示消息为广播的第一条消息；"id=A"表示消息的发送者为节点A。右边是slave节点记录的状态信息。

![](7.jpg)



为什么所有的节点都需要缓存一份发布的消息呢？

master发布的消息是依次经过所有slave节点，在这期间的任何时刻，有可能有节点失效，那么相邻的节点可能需要重新发送给新的节点。例如，A->B->C->D->A形成的循环链表，A为master节点，广播消息发送给节点B，B再发送给C，如果节点C收到B发送的消息还未发送给D时异常结束了，那么节点B感知后节点C失效后需要重新将消息发送给D。同样，如果B节点将消息发送给C后，B,C节点中新增了E节点，那么B节点需要再将消息发送给新增的E节点。

#### 1.3.2.5 消息的同步

配置镜像队列的时候有个`ha-sync-mode`属性，这个有什么用呢?

新节点加入到group后，最多能从左边节点获取到当前正在广播的消息内容，加入group之前已经广播的消息则无法获取到。如果此时master节点不幸失效，而新节点有恰好成为了新的master，那么加入group之前已经广播的消息则会全部丢失。

注意：这里的消息具体是指新节点加入前已经发布并复制到所有slave节点的消息，并且这些消息还未被消费者消费或者未被消费者确认。如果新节点加入前，所有广播的消息被消费者消费并确认了，master节点删除消息的同时会通知slave节点完成相应动作。这种情况等同于新节点加入前没有发布任何消息。

避免这种问题的解决办法就是对新的slave节点进行消息同步。当`ha-sync-mode`配置为自动同步（automatic）时，新节点加入group时会自动进行消息的同步；如果配置为manually则需要手动操作完成同步。

# 2、Federation

Federation直译过来是联邦，它的设计目标是使 RabbitMQ 在不同的 Broker 节点之间进行消息传递而无须建
立集群。具有以下特点：

- 支持不同管理域(不同的用户和vhost、不同版本的RabbitMQ)中的Broker或集群间传递消息
- 基于AMQP 0-9-1协议在不同的Broker之间通信，能容忍不稳定的网络连接情况

那么它到底有什么用呢？我们可以从一个实际场景入手：

有两个服务分别部署在国内和海外，它们之间需要通过消息队列来通讯。

很明显无论RabbitMQ部署在海外还是国内，另一方一定得忍受连接上的延迟。因此我们可以在海外和国内各部署一个MQ，这样一来海外连接海外的MQ,国内连接国内，就不会有连接上的延迟了。

但这样还会有问题，假设某生产者将消息存入海外MQ中的某个队列 queueB ， 在国内的服务想要消费 queueB  消息，消息的流转及确认必然要忍受较大的网络延迟 ，内部编码逻辑也会因这一因素变得更加复杂。

此外，服务可能得维护两个MQ的配置，比如国内服务在生产消息时得使用国内MQ，消费消息时得监听海外MQ的队列，降低了系统的维护性。

可能有人想到可以用集群，但是RabbitMQ的集群对延迟非常敏感，一般部署在局域网内，如果部署在广域网可能会产生网络分区等等问题。

这时候，Federation就派上用场了。它被设计成能够容忍不稳定的网络连接情况，完全能够满足这样的场景。

## 2.1 联邦交换器

那使用Federation之后是怎样的业务流程呢?

首先我们在海外MQ上定义exchangeA，它通过路由键“rkA”绑定着queueA。然后用Federation在exchangeA上建立一条**单向**连接到国内RabbitMQ，Federation则自动会在国内RabbitMQ建立一个exchangeA交换器（默认同名）。

![](8.png)

这时候，如果部署在国内的client C在国内MQ上publish了一条消息，这条消息会通过 Federation link 转发到海外MQ的交换器exchangeA中，最终消息会存入与 exchangeA 绑定的队列 queueA 中，而client C也能立即得到返回。

实际上，Federation插件还会在国内MQ建立一个内部的交换器：exchangeA→ broker3 B（broker3是集群名），并通过路由键 "rkA"将它和国内MQ的exchangeA绑定起来。接下来还会在国内MQ上建立一个内部队列federation: exchangeA->broker3 B，并与内部exchange绑定。这些操作都是内部的，对客户端来说是透明的。

值得一提的是，Federation的连接是单向的，如果是在海外MQ的exchangeA上发送消息是不会转到国内的。

这种在exchange上建立连接进行联邦的，就叫做**联邦交换器**。一个联邦交换器接收上游（upstream）的信息，这里的上游指的是其他的MQ节点。

对比前面举的例子，国内MQ就是上游，联邦交换器能够将原本发送给上游交换器的消息路由到本地的某个队列中。

## 2.2 联邦队列

有联邦交换器自然也有联播队列，联邦队列则允许一个本地消费者接收到来自上游队列的消息 。

![](9.png)

如图，海外MQ有队列A，给其设置一条链接，Federation则自动会在国内RabbitMQ建立一个队列A（默认同名）。

当有消费者 ClinetA连接海外MQ并消费 queueA 中的消息时，如果队列 queueA中本身有若干消息堆积，那么 ClientA直接消费这些消息，此时海外MQ中的queueA并不会拉取国内中的 queueA 的消息；如果队列 queueA中没有消息堆积或者消息被消费完了，那么它会通过 Federation link 拉取上游队列 queueA 中的消息(如果有消息)，然后存储到本地，之后再被消费者 ClientA进行消费 。

## 2.3 Federation使用

首先开启Federation 功能：

```shell
##启用插件
rabbitmq-plugins enable rabbitmq_federation
##启用管理插件
rabbitmq-plugins enable rabbitmq_federation_management
```

值得注意的是，当需要在集群中使用 Federation 功能的时候，集群中所有的节点都应该开启 Federation 插件。

接下来我们要配置两个东西：upstreams和Policies。

每个 upstream 用于定义与其他 Broker 建立连接的信息。

![](10.png)

通用参数如下：

- `name`: 定义这个upstreams的名称
- `URI` : 定义 upstreams的 AMQP 连接。例如`amqp://username:password@server-name/my-vhost`
- `Prefetch count` : 定义 Federation 内部缓存的消息条数，即在收到上游消息之后且在发送到下游之前缓存的消息条数。
- `Reconnect delay`: Federation link 由于某种原因断开之后，需要等待多少秒开始重新建立连接。
- `Acknowledgement Mode`: 定义 Federation link 的消息确认方式 。其有 3 种: on-confirm、 on-publish 、 no-acko 默认为 on-confirm，表示在接收到下游的确认消息之后再向上游发送消息确认，这个选项可以确保网络失败或者 Broker 密机时不会丢失消息，但也是处理速度最慢的选项。如果设置为 on-publish ，则表示消息发送到下游后(井需要等待下游的 Basic . Ack)再向上游发送消息确认，这个选项可以确保在网络失败的情况下不会丢失消息，但不能确保 Broker 岩机时不会丢失消息。 no-ack 表示无须进行消息确认，这个选项处理速度最快，但也最容易丢失消息。
- `Expires`：连接断开后，上游队列的超时时间，默认为none，表示不删除，单位为ms。相当于设置队列的x-expires参数，设置该值可以避免连接断开后，生产者一直向上游队列发送消息，造成上游大量消息堆积。

然后定义一个 Policy， 用于匹配交换器：

![](11.png)

`^exchange`意思是将匹配所有以exchange名字开头的交换器，为它们在上游创建连接。这样就创建了一个 Federation link。

# 3、Shovel

Shovel是RabbitMQ的一个插件， 能够可靠、持续地从一个Broker 中的队列（作为源端，即source ）拉取数据并转发至另一个Broker 中的交换器（作为目的端，即destination ）。作为源端的队列和作为目的端的交换器可以同时位于同一个 Broker 上，也可以位于不同的 Broker 上。

使用Shovel有以下优势：

- 松耦合，解决不同Broker、集群、用户、vhost、MQ和Erlang版本之间的消息移动
- 支持广域网，基于 AMQP 0-9-1 协议实现，可以容忍糟糕的网络，允许连接断开的同时不丢失消息
- 高度定制，当Shovel成功连接后，可以配置

使用Shovel时，通常源为队列，目的为交换器：

![](12.jpeg)

但是，也可以源为队列，目的为队列。实际也是由交换器转发，只不过这个交换器是默认交换器。配置交换器做为源也是可行的。实际上会在源端自动新建一个队列，消息先存在这个队列，再被Shovel移走。

使用Shovel插件命令：

```shell
##启用插件
rabbitmq-plugins enable rabbitmq_shovel
##启用管理插件
rabbitmq-plugins enable rabbitmq_shovel_management
```

Shovel 既可以部署在源端，也可以部署在目的端。有两种方式可以部署 Shovel：

- 静态方式：在 `rabbitmq.config` 配置文件中设置
- 动态方式：通过 Runtime Parameter 设置

其主要差异如下：

| Static Shovels                                               | Dynamic Shovels                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 基于 broker 的配置文件进行定义                               | 基于 broker 的 parameter 参数进行定义                        |
| 需要重启宿主 broker 以便配置生效                             | 可以在任意时间进行创建和删除，直接生效                       |
| 更加通用：任何 queue 、exchange 或 binding 关系均可在启动时手动声明 | 更具有目标性：被 shovel 所使用的 queue 、exchange 和 binding 关系能够自动被声明 |

来看一个使用Shovel治理消息堆积的案例。

当某个队列中的消息堆积严重时，比如超过某个设定的阈值，就可以通过 Shovel 将队列中的消息移交给另一个集群。

![](13.png)

- 情形 1：当检测到当前运行集群 cluster1 中的队列 queue1 中有严重消息堆积，比如超过2 千万或者消息占用大小(messages bytes) 超过10GB 时，就启用 shovel1 将队列 queue1 中的消息转发至备份集群 cluster2 中的队列queue2 。
- 情形 2 ：紧随情形1，当检测到队列queue1 中的消息个数低于1 百万或者消息占用大小低于1GB 时就停止shovel1 ，然后让原本队列 queue1 中的消费者慢慢处理剩余的堆积。
- 情形 3：当检测到队列 queue1 中的消息个数低于10 万或者消息占用大小低于100MB时，就开启 shovel2 将队列 queue2 中暂存的消息返还给队列queue1 。
- 情形 4：紧随情形3 ，当检测到队列queuel 中的消息个数超过 1百万或者消息占用大小高于1GB 时就将shovel2 停掉。