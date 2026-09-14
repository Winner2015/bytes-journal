# 1、概述

在现代的企业环境中，单机容量往往无法存储大量数据，需要跨机器存储。统一管理分布在集群上的文件系统称为**分布式文件系统**。

![](1.png)

**HDFS**（Hadoop Distributed File System）是 Hadoop  的核心组件之一， 非常适于存储大型数据 (比如 TB 和 PB)， HDFS 使用多台计算机存储文件，并且提供统一的访问接口，像是访问一个普通文件系统一样使用分布式文件系统。

HDFS是分布式计算中数据存储管理的基础，是基于流数据模式访问和处理超大文件的需求而开发的，可以运行于廉价的商用服务器上。它所具有的**高容错、高可靠性、高可扩展性、高获得性、高吞吐率**等特征为海量数据提供了不怕故障的存储，为超大数据集的应用处理带来了很多便利。

HDFS 具有以下**优点**：

- 1、高容错性

  - 数据自动保存多个副本。它通过增加副本的形式，提高容错性。
  - 某一个副本丢失以后，它可以自动恢复，这是由 HDFS 内部机制实现的，我们不必关心。

- 2、适合批处理

  - 通过移动计算而不是移动数据。

  - 它会把数据位置暴露给计算框架。

- 3、适合大数据处理

  - 处理数据达到 GB、TB、甚至PB级别的数据。
  - 能够处理百万规模以上的文件数量，数量相当之大。
  - 能够处理10K节点的规模。

- 4、流式文件访问

  - 一次写入，多次读取。**文件一旦写入不能修改，只能追加**。
  - 它能保证数据的一致性

- 5、可构建在廉价机器上

  - 它通过多副本机制，提高可靠性。
  - 它提供了容错和恢复机制。比如某一个副本丢失，可以通过其它副本来恢复。

当然 HDFS 也有它的**劣势**，并不适合以下场合：

- 1、低延时数据访问
  - 比如毫秒级的来存储数据，这是不行的，它做不到。
  - 它适合高吞吐率的场景，就是在某一时间内写入大量的数据。但是它在低延时的情况下是不行的，比如毫秒级以内读取数据，这样它是很难做到的。
- 2、小文件存储
  - 存储大量小文件(这里的小文件是指小于HDFS系统的Block大小的文件（默认128M）)的话，它会占用 NameNode大量的内存来存储文件、目录和块信息。这样是不可取的，因为NameNode的内存总是有限的。
  - 小文件存储的寻道时间会超过读取时间，它违反了HDFS的设计目标。
- 3、并发写入、文件随机修改
  - 一个文件只能有一个写，不允许多个线程同时写。
  - 仅支持数据 append（追加），不支持文件的随机修改。

# 2、架构

![](2.png)

HDFS 采用Master/Slave的架构来存储数据，这种架构主要由四个部分组成，分别为HDFS Client、NameNode、DataNode和Secondary NameNode。

## 2.1 NameNode

Namenode是整个文件系统的管理节点，负责接收用户的操作请求。它维护着整个文件系统的目录树，文件的元数据信息以及文件到块的对应关系和块到节点的对应关系。

Namenode保存了两个核心的数据结构：

- **FsImage：**FsImage是NameNode内存中元数据的镜像文件，是元数据的一个永久性checkpoint，包含了HDFS的所有目录和文件idnode的序列化信息,可以类比银行的账户余额,只有简单的信息。
- **EditLog：**EditLog是用于衔接内存元数据和FsImage之间的操作日志，保存了自最后一次检查点之后，所有针对HDFS文件系统的操作，比如增加文件、重命名文件、删除目录等等，可以类比银行的账户流水，包括每一笔的记录，如果日积月累，流水信息可以非常大。

在NameNode启动的时候，先将fsimage中的文件系统元数据信息加载到内存，然后根据edits中的记录将内存中的元数据同步到最新状态；所以，这两个文件一旦损坏或丢失，将导致整个HDFS文件系统不可用。

为了避免edits文件过大，**SecondaryNameNode会按照时间阈值或者大小阈值，周期性的将fsimage和edits合并**，然后将最新的fsimage推送给NameNode。

## 2.2 SecondaryNameNode

并非 NameNode 的热备。当NameNode 挂掉的时候，它并不能马上替换 NameNode 并提供服务。其主要任务是辅助 NameNode，定期合并 fsimage和fsedits。

## 2.3 DataNode

Datanode是实际存储数据块的地方，负责执行数据块的读/写操作。

一个数据块在DataNode以文件存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据，包括数据块的长度，块数据的校验和，以及时间戳。

文件划分成块，默认大小128M，以快为单位，每个块有多个副本（默认3个）存储不同的机器上。

Hadoop2.X默认128M，**小于一个块的文件，并不会占据整个块的空间**。Block数据块大小设置较大的原因：

- 减少文件寻址时间
- 减少管理快的数据开销，因每个块都需要在NameNode上有对应的记录
- 对数据快进行读写，减少建立网络的连接成本

## 2.4 Client

文件上传 HDFS 的时候，Client 将文件切分成 一个一个的Block，然后进行存储。

Client 还提供一些命令来管理 HDFS，比如启动或者关闭HDFS。

# 3、实现原理

## 3.1 读流程

![](3.png)

1. 客户端通过调用FileSystem对象中的open()方法来读取需要的数据。
2. DistributedFileSystem通过RPC(远程过程调用)获得文件的第一批block的locations，同一block按照重复数会返回多个locations，这些locations按照hadoop拓扑结构排序，距离客户端近的排在前面。
3. 客户端调用read方法，DFSInputStream就会找出离客户端最近的datanode并连接datanode。
4. 数据从datanode源源不断的流向客户端。
5. 如果第一个block块的数据读完了，就会关闭指向第一个block块的datanode连接，接着读取下一个block块。这些操作对客户端来说是透明的，从客户端的角度来看只是读一个持续不断的流。
6. 如果第一批block都读完了，DFSInputStream就会去namenode拿下一批blocks的location，然后继续读，如果所有的block块都读完，这时就会关闭掉所有的流。

## 3.2 写流程

![](4.png)

1. 客户端通过调用 DistributedFileSystem 的create方法，创建一个新的文件。
2. DistributedFileSystem 通过 RPC（远程过程调用）调用 NameNode，去创建一个没有blocks关联的新文件。创建前，NameNode 会做各种校验，比如文件是否存在，客户端有无权限去创建等。如果校验通过，NameNode 就会记录下新文件，否则就会抛出IO异常。
3. 客户端开始写数据到DFSOutputStream,DFSOutputStream会把数据切成一个个小packet，然后排成队列 **data queue**。
4. DataStreamer 会去处理接受 data queue，它先问询 NameNode 这个新的 block 最适合存储的在哪几个DataNode里，比如重复数是3，那么就找到3个最适合的 DataNode，把它们排成一个 **pipeline**。DataStreamer 把 packet 按队列输出到管道的第一个 DataNode 中，第一个 DataNode又把 packet 输出到第二个 DataNode 中，以此类推。
5. DFSOutputStream 还有一个队列叫 **ack queue**，也是由 packet 组成，等待DataNode的收到响应，当pipeline中的所有DataNode都表示已经收到的时候，这时akc queue才会把对应的packet包移除掉。
6. 客户端完成写数据后，调用close方法关闭写入流。
7. DataStreamer 把剩余的包都刷到 pipeline 里，然后等待 ack 信息，收到最后一个 ack 后，通知 DataNode 把文件标示为已完成。

## 3.3 checkpoint机制

Namenode始终在内存中保存metedata，用于处理“读请求”，到有“写请求”到来时，namenode会首**先写editlog到磁盘，即向edits文件中写日志，成功返回后，才会修改内存**，并且向客户端返回，Hadoop会维护一个fsimage文件，也就是namenode中metedata的镜像，但是fsimage不会随时与namenode内存中的metedata保持一致，而是每隔一段时间通过合并edits文件来更新内容。

![](5.png)

1. 将hdfs更新记录写入一个新的文件——edits.new
2. 将fsimage和edits文件通过http协议发送至secondary namenode
3. 将fsimage和edits合并，生成一个新的文件fsimage.ckpt。
4. 将生成的fsimage.ckpt文件通过http协议发送至NameNode
5. 重命名fsimage.ckpt为fsimage，edits.new为edits

## 3.4 HA方案

### 3.4.1 热备份

HDFS HA（High Availability）是为了解决单点故障问题。

HA集群设置两个名称节点，“活跃（**Active**）”和“待命（**Standby**）”，两种名称节点的状态同步，可以借助于一个共享存储系统来实现，一旦活跃名称节点出现故障，就可以立即切换到待命名称节点。

![](6.png)

为了保证读写数据一致性，HDFS集群设计为只能有一个状态为Active的NameNode，但这种设计存在单点故障问题，官方提供了两种解决方案：

- **QJM**（推荐）：通过同步编辑事务日志的方式备份命名空间数据，同时需要DataNode向所有NameNode上报块列表信息。还可以配置ZKFC组件实现故障自动转移。
- **NFS**：将需要持久化的数据写入本地磁盘的同时写入一个远程挂载的网络文件系统做为备份。

通过增加一个Secondary NameNode节点，处于Standby的状态，与Active的NameNode同时运行。当Active的节点出现故障时，切换到Secondary节点。

为了保证Secondary节点能够随时顶替上去，Standby节点需要定时同步Active节点的事务日志来更新本地的文件系统目录树信息，同时DataNode需要配置所有NameNode的位置，并向所有状态的NameNode发送块列表信息和心跳。

同步事务日志来更新目录树由JournalNode的守护进程来完成，简称为QJM，一个NameNode对应一个QJM进程，当Active节点执行任何命名空间文件目录树修改时，它会将修改记录持久化到大多数QJM中，Standby节点从QJM中监听并读取编辑事务日志内容，并将编辑日志应用到自己的命名空间。发生故障转移时，Standby节点将确保在将自身提升为Active状态之前，从QJM读取所有编辑内容。

注意，QJM只是实现了数据的备份，当Active节点发送故障时，需要手工提升Standby节点为Active节点。如果要实现NameNode故障自动转移，则需要配套ZKFC组件来实现，ZKFC也是独立运行的一个守护进程，基于zookeeper来实现选举和自动故障转移。



### 3.4.2 HDFS联邦（Federation）

虽然HDFS HA解决了“单点故障”问题，但是在系统扩展性、整体性能和隔离性方面仍然存在问题：

1. 系统扩展性方面，元数据存储在NN内存中，受内存上限的制约。
2. 整体性能方面，吞吐量受单个NN的影响。
3. 隔离性方面，一个程序可能会影响其他运行的程序，如一个程序消耗过多资源导致其他程序无法顺利运行。

HDFS HA本质上还是单名称节点。HDFS联邦可以解决以上三个方面问题。

![](7.png)

在HDFS联邦中，设计了多个相互独立的NN，使得HDFS的命名服务能够水平扩展，这些NN分别进行各自命名空间和块的管理，不需要彼此协调。每个DN要向集群中所有的NN注册，并周期性的发送心跳信息和块信息，报告自己的状态。

HDFS联邦拥有多个独立的命名空间，其中，每一个命名空间管理属于自己的一组块，这些属于同一个命名空间的块组成一个“块池”。每个DN会为多个块池提供块的存储，块池中的各个块实际上是存储在不同DN中的。

