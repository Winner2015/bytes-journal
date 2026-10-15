# 1 RabbitMQ 是什么

RabbitMQ 是一个开源的 AMQP 实现，服务器端用 Erlang 语言编写，支持多种客户端。它最初起源于金融系统，用于在分布式系统中存储转发消息。

RabbitMQ 在易用性、扩展性、高可用性等方面表现不俗。具体特点包括：

- **可靠性**（Reliability）：RabbitMQ 使用一些机制来保证可靠性，如持久化、传输确认、发布确认。
- **灵活的路由**（Flexible Routing）：在消息进入队列之前，通过 Exchange 来路由消息的。对于典型的路由功能，RabbitMQ 已经提供了一些内置的 Exchange 来实现。针对更复杂的路由功能，可以将多个 Exchange 绑定在一起，也通过插件机制实现自己的 Exchange。
- **消息集群**（Clustering）：多个 RabbitMQ 服务器可以组成一个集群，形成一个逻辑 Broker。
- **高可用**（Highly Available Queues）：队列可以在集群中的机器上进行镜像，使得在部分节点出问题的情况下队列仍然可用。
- **多种协议**（Multi-protocol）：RabbitMQ 除了原生支持 AMQP 协议，还支持 STOMP、MQTT 等多种消息中间件协议。
- **多语言客户端**（Many Clients）：RabbitMQ 几乎支持所有常用语言，比如 Java、.NET、Ruby 等等。
- **管理界面**（Management UI）：RabbitMQ 提供了一个易用的用户界面，使得用户可以监控和管理消息 Broker 的许多方面。
- **跟踪机制**（Tracing）：如果消息异常，RabbitMQ 提供了消息跟踪机制，使用者可以找出发生了什么。
- **插件机制**（Plugin System）：RabbitMQ 提供了许多插件，来从多方面进行扩展，也可以编写自己的插件。

# 2 基本概念

![](imgs/1.png)

- **生产者**（producer）：创建消息，发布到代理服务器（Message Broker）。
- **消费者**（consumer）：连接到代理服务器，并订阅到队列（queue）上，代理服务器将发送消息给一个订阅的/监听的消费者。

- **消息**（Message）：包含消息头（即附属的配置信息）和消息体（即消息的实体内容）。消息体也可以称为 payload，消息体是不透明的，而消息头则由一系列的可选属性组成，这些属性包括 routing-key（路由键）、priority（相对于其他消息的优先权）、delivery-mode（指出该消息可能需要持久性存储）等。生产者把消息交由 RabbitMQ 后，RabbitMQ 会根据消息头把消息发送给感兴趣的 Consumer（消费者）。
- **交换器**（exchange）：在 RabbitMQ 中，消息并不是直接被投递到队列中的，中间还必须经过交换器这一层，交换器会把我们的消息分配到对应的队列中。交换器用来接收生产者发送的消息并将这些消息路由给服务器中的队列中，如果路由不到，或许会返回给生产者，或许会被直接丢弃掉。RabbitMQ 的交换器有 4 种类型，不同的类型对应着不同的路由策略：direct（默认）、fanout、topic 和 headers，不同类型的 Exchange 转发消息的策略有所区别。
- **队列**（queue）：用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。RabbitMQ 中消息只能存储在队列中，这一点和 Kafka 相反。Kafka 将消息存储在 topic（主题）这个逻辑层面，而相对应的队列逻辑只是 topic 实际存储文件中的位移标识。RabbitMQ 的生产者生产消息并最终投递到队列中，消费者可以从队列中获取消息并消费。多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理，这样避免了消息被重复消费。RabbitMQ 不支持队列层面的广播消费，如果有广播消费的需求，需要在其上进行二次开发。

- **路由键**（routing key）：消息头的一个属性（默认为空），交换器根据这个路由键将消息发送到匹配的队列中。

- **绑定键**（binding key）：绑定就是基于 Binding Key 将 Exchange 和 Queue 连接起来的路由规则，所以可以将交换器理解成一个由 Binding 构成的路由表。队列需要通过绑定键（默认为空）绑定到交换器上，交换器将消息的路由键与所绑定队列的绑定键进行匹配，正确匹配的消息将发送到队列中。路由键是偏向生产的概念，而绑定键是偏向消费的概念。

- **代理服务器**（Message Broker）：对于 RabbitMQ 来说，一个 RabbitMQ Broker 可以简单地看作一个 RabbitMQ 服务节点，或者 RabbitMQ 服务实例。大多数情况下也可以将一个 RabbitMQ Broker 看作一台 RabbitMQ 服务器。

- **虚拟主机**（vhost）：AMQP 概念的基础，其本质上就是一个 mini 版的代理服务器（拥有自己的队列、交换器和绑定，更重要的是，拥有自己的权限机制），RabbitMQ 默认的 vhost：“/”（类似于网络中的 namespace），每个用户只能访问自己的 vhost（通常会被指派至少一个 vhost），进而用户只能访问自己的队列、交换器和绑定，所以 vhost 之间是绝对隔离的。
- **信道**（channel）：应用程序（生产与/或消费）和代理服务器之间 TCP 连接内的虚拟连接，解决 TCP 连接数量限制及降低 TCP 连接代价。每个信道有一个 ID，其概念与“频分多路复用”类似。

# 3 路由策略

路由策略是指 Exchange 在收到生产者发送的消息后依据什么规则把消息转发到一个或多个队列中保存。路由策略与三个因素相关：

- Exchange Type（Exchange 的类型）
- Binding Key（Exchange 和 Queue 的绑定关系）
- Routing Key & headers（消息的标记信息）

Exchange 根据消息的 Routing Key 和 Exchange 绑定 Queue 的 Binding Key 分配消息。生产者在将消息发送给 Exchange 的时候，一般会指定一个 Routing Key，来指定这个消息的路由规则，而这个 Routing Key 需要与 Exchange Type 及 Binding Key 联合使用才能最终生效。
在Exchange Type与Binding Key固定的情况下（一般这些内容都是固定配置好的），我们的生产者就可以在发送消息给Exchange时，通过指定Routing Key来决定消息流向哪里。

## 3.1 Direct 策略

![](imgs/2.png)

当消息的 Routing Key 与 Exchange 和 Queue 之间的 Binding Key 完全匹配时，才会将消息分发到该 Queue。

Direct是Exchange的**默认模式**。

RabbitMQ 默认提供了一个 Exchange，名字是空字符串，类型是 Direct，绑定到所有的 Queue（每一个 Queue 和这个无名 Exchange 之间的 Binding Key 是 Queue 的名字）。所以有时候我们感觉不需要交换器也可以发送和接收消息，但是实际上是使用了 RabbitMQ 默认提供的 Exchange。

![](imgs/3.jpg)

Exchange 和两个队列绑定在一起：

- Q1 的 binding key 是 orange
- Q2 的 binding key 是 black 和 green。
- 当 Producer publish 的 key 是 orange 时，exchange 会把它放到 Q1 上；如果是 black 或 green 就会到 Q2 上；其余的 Message 被丢弃

## 3.2 Topic 策略

![](imgs/4.png)

用消息的 Routing Key 与 Exchange 和 Queue 之间的 Binding Key 进行模糊匹配，如果匹配成功，将消息分发到该 Queue。

Binding Key 中可以存在两种特殊字符用于做模糊匹配：

- `*`：用于匹配一个单词
- `#`：用于匹配多个单词（可以是零个）

![](imgs/5.jpg)

Producer 发送消息时需要设置 routing_key。

Q1 的 binding key 是`*.orange.*`，Q2 是 `*.*.rabbit` 和 `lazy.#`。

产生一个 `test.orange.mm` 消息，则会路由到 Q1；如果是 `test.orange` 则无法路由到 Q1，因为 Q1 的规则是三个单词，中间一个为 orange，不满足这个规则的都无效。

产生一个 `test.qq.rabbit` 或者 `lazy.qq` 都可以分发到 Q2；即路由 key 为三个单词，最后一个为 rabbit 或者不限制单词个数，只要第一个是 lazy 的消息，都可以分发过来。

如果产生的是一个 `test.orange.rabbit` 消息，则 Q1 和 Q2 都可以满足。

## 3.3 Fanout 策略

![](imgs/6.png)

与 Binding Key 和 Routing Key 无关，交换器会把所有发送到该交换器的消息路由到所有与该交换器绑定的消息队列中。订阅模式。类似于子网广播，子网内的每台主机都获得了一份复制的消息。Fanout 交换机转发消息是最快的。

![](imgs/7.png)

## 3.4 Headers 策略

不依赖于 Routing Key 与 Binding Key，而是通过消息头的 Headers 属性来进行匹配转发的，类似 HTTP 请求的 Headers。

在绑定 Queue 与 Exchange 时指定一组键值对，键值对的 Hash 结构中要求携带一个键“x-match”，这个键的 Value 可以是 any 或 all，代表消息携带的 Hash 是需要全部匹配（all），还是仅匹配一个键（any）。

当消息发送到 Exchange 时，交换器会取到该消息的 headers，对比其中的键值对是否完全匹配 Queue 与 Exchange 绑定时指定的键值对；如果完全匹配则消息会路由到该 Queue，否则不会路由到该 Queue。Headers 交换机的优势是匹配的规则不被限定为字符串（String），而是 Object 类型。

![](imgs/8.png)

不过，在实际应用当中，路由策略的选择范围以 Direct、Topic、Fanout 为主，Headers 策略很少使用。
