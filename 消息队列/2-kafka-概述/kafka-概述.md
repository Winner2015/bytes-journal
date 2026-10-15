# 1 Kafka 是什么

Kafka 最早是由 LinkedIn 公司采用 Scala 和 Java 语言开发的一个多分区、多副本的分布式消息系统，现已成为 Apache 的顶级项目。早期版本依赖 ZooKeeper 进行集群协调，而现代版本已改用内置的 KRaft 模式。

Apache Kafka 与传统消息系统相比，有以下特点：

- 同时为发布和订阅提供高吞吐量
  Kafka 的设计目标是以时间复杂度为 O(1) 的方式提供消息持久化能力，即使对 TB 级以上数据也能保证常数时间的访问性能。即使在普通的商用机器上，经过合理配置后单机也能做到每秒数百万条消息的传输，集群整体吞吐量可达千万级别。
- 消息持久化
  将消息持久化到磁盘，因此可用于批量消费，例如 ETL 以及实时应用程序。通过将数据持久化到硬盘以及 replication 防止数据丢失。
- 分布式
  支持 Server 间的消息分区及分布式消费，同时保证每个 Partition 内的消息顺序传输。这样易于向外扩展，所有的 Producer、Broker 和 Consumer 都会有多个，均为分布式的。无需停机即可扩展机器。
- 消费消息采用 pull 模式
  消息被处理的状态是在 Consumer 端维护，而不是由 Server 端维护，Broker 无状态，Consumer 自己保存 offset。

如今，Apache Kafka 的定位早已超越了传统的消息中间件，演变成了一个完整的分布式事件流处理平台。除了核心的发布订阅功能外，它还原生集成了 Kafka Connect（用于与外部系统进行大规模数据导入导出）以及 Kafka Streams（用于构建轻量级的实时流处理应用）。这种一体化的设计使得企业能够在一个统一的平台上完成数据的采集、存储、处理和路由，极大地简化了现代数据架构的复杂度。

# 2 体系架构

![](imgs/1.jpeg)

- **Producer**
  通过 push 模式向 Kafka Broker 发送消息。发送的消息可以是网站的页面访问、服务器日志，也可以是 CPU 和内存相关的系统资源信息。
- **Broker**
  用于存储消息的服务器，可以简单地看作一个独立的 Kafka 实例。Kafka Broker 支持水平扩展。Kafka Broker 节点的数量越多，Kafka 集群的吞吐率越高。
- **Consumer**
  通过 pull 模式从 Broker 订阅并消费消息。每个 Consumer 属于一个特定的 Consumer Group（使用订阅模式时必须指定 Group Name，否则客户端会抛出异常；直接手动分配分区则不需要指定）。
- **ZooKeeper**
  老版本中负责存储集群元数据、协助 Controller 选举等，这部分职责在现代 Kafka 中已由集群内部的 KRaft Controller 节点接管；而 Consumer Group 的重平衡则由 Broker 端的 Group Coordinator 统一调度。

值得注意的是，Kafka 的体系架构在近年来经历了重大演进。为了解决 ZooKeeper 带来的运维复杂性以及元数据同步的性能瓶颈，社区引入了基于 Raft 协议的 KRaft（Kafka Raft Metadata mode）模式。在 KRaft 架构下，集群不再依赖外部协调服务，而是由专门的 Controller 节点组成 Quorum 来管理元数据。随着 2025 年 3 月 Kafka 4.0.0 版本的正式发布，对 ZooKeeper 的支持已被彻底移除，KRaft 成为唯一的标准运行模式；这不仅让集群部署更加轻量化，还大幅提升了可支持的分区数量上限以及 Controller 的故障恢复速度。

# 3 Topic 与 Partition

一个消息中间件，队列不单单只有一个，我们往往会有多个队列，而我们生产者和消费者就得知道：将数据发送到哪个队列，从哪个队列取消息。我们需要给队列取名字，叫做 **Topic**：

![](imgs/2.png)

为了提高一个 Topic 的吞吐量，Kafka 会对 Topic 进行**分区**（Partition）。所以，生产者实际上是往 Topic 的一个 Partition 发送数据，而消费者实际上也是从一个 Topic 的 Partition 拉取数据：

![](imgs/3.png)

Kafka 可以保证**单个 Partition 的写入是有顺序的**。如果要保证全局有序，那只能写入一个 Partition 中。如果要消费也有序，消费者也只能有一个。

凡是分布式就无法避免网络抖动/机器宕机等问题的发生，很有可能消费者 A 读取了数据，还没来得及消费，就宕机了。Broker 端的 Group Coordinator 通过心跳机制发现消费者 A 已宕机，随后触发消费者组的 Rebalance（重平衡），让消费者 B 去消费原本属于消费者 A 的分区，等消费者 A 重连的时候，会发现同一条数据已经被重复消费了。如果业务上不允许重复消费，最好在消费者端做业务上的校验。

一个 Broker 就是一个 Kafka 服务器，多个 Broker 构成一个 Kafka 集群。一个 Topic 会分为多个 Partition，Partition 会分布在不同的 Broker 中。

也就是说，**Kafka 是天然分布式的**。

# 4 多副本机制

既然是分布式，肯定会有单点问题：如果其中一台 Broker 宕机了，怎么办？

Kafka 为分区引入了**多副本**（Replica）机制，通过增加副本数量可以提升容灾能力。比如，现在我们有三个 Partition，分别存在三台 Broker 上。每个 Partition 都会备份，这些备份散落在不同的 Broker 上：

![](imgs/4.png)

图中红色的 Partition 代表 Leader 副本（即**主分区**），紫色的代表 Follower 副本（即**备份分区**）。生产者往 Topic 写数据，是与主分区交互，消费者消费 Topic 的数据，也是与主分区交互。

**备份分区仅仅用作备份，不做读写**。如果某个 Broker 宕机，那就会选举出其他 Broker 的 Partition 来作为主分区，这就实现了高可用。

分区中的所有副本统称为 **AR**（Assigned Replicas）。所有与 Leader 副本保持一定程度同步的副本（包括 Leader 副本在内）组成 **ISR**（In-Sync Replicas），ISR 集合是 AR 集合中的一个子集。消息会先发送到 Leader 副本，然后 Follower 副本才能从 Leader 副本中拉取消息进行同步，同步期间内 Follower 副本相对于 Leader 副本而言会有一定范围的滞后，这个范围可以通过参数进行配置。与 Leader 副本同步滞后过多的副本（不包括 Leader 副本）组成 **OSR**（Out-of-Sync Replicas），由此可见：`AR=ISR+OSR`。在正常情况下，所有的 Follower 副本都应该与 Leader 副本保持一定程度的同步，即 AR=ISR，OSR 集合为空。

由此可见，Kafka 的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的 Follower 副本都复制完，这条消息才会被确认为已成功提交，这种复制方式极大地影响了性能。而在异步复制方式下，Follower 副本异步地从 Leader 副本中复制数据，数据只要被 Leader 副本写入就被认为已经成功提交。在这种情况下，如果 Follower 副本都还没有复制完而落后于 Leader 副本，突然 Leader 副本宕机，则会造成数据丢失。Kafka 使用的这种 ISR 的方式则有效地权衡了数据可靠性和性能之间的关系。

ISR 与 **HW** 和 **LEO** 也有紧密的关系。HW 是 High Watermark 的缩写，俗称**高水位**，它标识了一个特定的消息偏移量（offset），消费者只能拉取到这个 offset 之前的消息。

如下图所示，它代表一个日志文件，这个日志文件中有 9 条消息，第一条消息的 offset 为 0，最后一条消息的 offset 为 8，offset 为 9 的消息用虚线框表示，代表下一条待写入的消息。日志文件的 HW 为 6，表示消费者只能拉取到 offset 在 0 至 5 之间的消息，而 offset 为 6 的消息对消费者而言是不可见的。

![](imgs/5.png)

LEO 是 Log End Offset 的缩写，它标识当前日志文件中下一条待写入消息的 offset，上图中 offset 为 9 的位置即为当前日志文件的 LEO，LEO 的大小相当于当前日志分区中最后一条消息的 offset 值加 1。分区 ISR 集合中的每个副本都会维护自身的 LEO，而 ISR 集合中最小的 LEO 即为分区的 HW，对消费者而言只能消费 HW 之前的消息。

为了保证消息的持久化，Kafka 会将 Partition 的数据写在磁盘（消息日志），不过 Kafka **只允许追加写入**（顺序访问），避免缓慢的随机 I/O 操作。当然，Kafka 也不是 Partition 一有数据就立马将数据写到磁盘上，它会先缓存一部分，等到足够多数据量或等待一定的时间再批量写入。

在保障数据可靠性的同时，Kafka 对数据一致性的追求也在持续演进。针对跨分区写入的事务场景，Kafka 4.0 落地了事务协议的升级（KIP-890，Transactions Server-Side Defense）：Broker 端会主动拒绝乱序或延迟的事务写入，避免悬挂事务（Hanging Transaction）长期不提交、阻塞 LSO（Last Stable Offset）并使 READ_COMMITTED 消费者无法继续读取；transactions v2 还修复了事务消息可能“泄漏”到其他事务的一致性缺陷。这些改进让开启 Exactly-Once 语义（EOS）的应用在面对频繁的 Broker 重启或网络分区时，也能获得更严格的一致性保证。

虽然本地磁盘的顺序写入保证了极高的性能，但随着业务对数据保留时长要求的不断增加，将所有历史消息都存储在 Broker 本地的高性能磁盘上会导致极高的硬件成本。为此，Kafka 在 3.6 版本中引入了分层存储（Tiered Storage，KIP-405）的早期预览，并在 3.9 版本中正式达到生产可用。该机制允许将较旧的、访问频率低的日志段自动卸载到成本更低廉的外部对象存储（如 Amazon S3、HDFS 或 MinIO）中，本地磁盘仅保留近期的热数据；当消费者需要读取冷数据时，Broker 会透明地从远程存储中获取。这一特性在不改变现有 API 的前提下实现了计算与存储的解耦，使得 Kafka 集群可以经济高效地保留数月甚至数年的海量数据。

# 5 消费者组

![](imgs/6.png)

生产者可以有多个，消费者也可以有多个。像上面图的情况，是一个消费者消费三个分区的数据。多个消费者可以组成一个**消费者组**。

- 如果消费者组中的某个消费者宕机了，那么存活的其中一个消费者可能就要消费两个 Partition 了

- 如果只有三个 Partition，而消费者组有 4 个消费者，那么一个消费者会空闲

- 消费者组之间从逻辑上是独立的，如果多加入一个消费者组，无论是新增的消费者组还是原本的消费者组，都能消费 Topic 的全部数据。

那么问题来了，如果一个消费者组中的某个消费者宕机了，存活的消费者需要知道它消费到哪了。

这里要引出**offset**了，offset 是消息在分区中的唯一标识，说白了就是表示消费者的消费进度。Kafka 通过 offset 来保证消息在分区内的顺序性，不过 offset 并不跨越分区，也就是说，**Kafka 保证的是分区有序而不是主题有序**。

![](imgs/7.png)

在 0.8.x 及更早的版本中，offset 是由 ZooKeeper 管理的。但由于 ZooKeeper 不适合高频的读写操作，从 0.9.0.0 版本开始，Kafka 将 offset 统一迁移到 Broker 端一个名为 `__consumer_offsets` 的内部 Topic 中，这一机制沿用至今。

每次消费者消费的时候，都会提交这个 offset，Kafka 可以让你选择是自动提交还是手动提交。

除了 offset 的提交方式，消费者组还有一个绕不开的运行机制——Rebalance（重平衡）：它往往会导致整个组短暂停止消费，这在大型集群中被称为“Stop-the-World”问题。为了缓解这一痛点，现代 Kafka 先后引入了两项重要优化。首先是增量协作式重平衡（Incremental Cooperative Rebalancing），它将过去“先全部撤销再重新分配”的激进策略改为“仅迁移必须迁移的分区”，从而保证大部分消费者在重平衡期间依然可以继续处理消息。其次是静态成员资格（Static Membership），通过为消费者分配固定的 `group.instance.id`，那些因短暂重启或发布而离开的消费者在超时时间内重新加入时，可以直接恢复原有的分区分配，避免了不必要的重平衡风暴。

更为突破性的变化是自 Kafka 4.0 起正式可用（GA）的新一代消费者重平衡协议（KIP-848）。在旧协议中，分区分配的计算由消费者端的 Group Leader 负责，一旦这个客户端出现网络抖动，就可能引发整个组的连锁重平衡；新协议把分配计算完全收归到 Broker 端的 Group Coordinator，消费者只需维持心跳并被动接收分配结果。这种服务端主导（Server-side Assignment）的架构不仅消除了客户端 Group Leader 失效引发的连锁影响，还把超大规模消费者组的重平衡耗时从分钟级压缩到了秒级。目前新协议尚未成为默认配置，可按消费者组逐步启用。
