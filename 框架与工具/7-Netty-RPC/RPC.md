# Netty RPC

# 1 RPC

在微服务大行其道的今天，分布式系统越来越重要，实现服务化首先就要考虑服务之间如何通信，也就是 **RPC**。

> RPC，全称为 Remote Procedure Call，即远程过程调用，它是一种计算机通信协议，允许像调用本地服务一样调用远程服务。

一次 RPC 调用过程可以简化为下图：

![](imgs/1.png)

- Client 会调用本地动态代理 Proxy
- 这个代理会按照协议将调用序列化为字节流
- 通过网络框架（比如 Netty）将字节流发送到服务端
- 服务端在收到这个字节流后，会根据协议将其反序列化为原始的调用，利用反射原理调用服务方提供的方法
- 如果请求有返回值，还需要把结果根据协议序列化后，再通过网络框架返回给调用方

RPC 框架有很多，比较知名的如阿里开源的 Dubbo（现为 Apache Dubbo）、Google 的 gRPC、Go 语言的 rpcx、Apache 的 Thrift。Spring Cloud 则是一套微服务工具集，服务间调用通常借助 OpenFeign 基于 HTTP 完成，严格来说并不是 RPC 框架。

RPC 框架让调用方像调用本地函数一样调用远程服务，那么如何做到这一点呢？

在使用的时候，调用方直接调用本地函数，传入相应参数，其他细节它不用管，通信细节交给 RPC 框架来实现。实际上 RPC 框架采用代理类的方式，具体来说是动态代理的方式，在运行时动态创建新的类，也就是代理类，在该类中实现通信的细节问题，比如参数序列化。

当然不光是序列化，我们还需要约定一个双方通信的协议格式，规定好请求参数的数据类型、请求的参数、请求的方法名等，这样根据格式进行序列化后进行网络传输，然后服务端收到请求对象后按照指定格式进行解码，这样服务端才知道具体该调用哪个方法、传入什么样的参数。

刚才又提到网络传输，RPC 框架重要的一环就是网络传输。服务是部署在不同主机上的，如何高效地进行网络传输，尽量不丢包，保证数据完整无误地快速传递出去？答案就是利用我们今天的主角——Netty。

再总结一下，一个 RPC 框架需要重点关注以下几点：

- 代理（动态代理）
- 通信协议
- 序列化
- 网络传输

在通信协议方面，传统 RPC 框架多是在 TCP 之上自定义私有协议，本文的实现也是如此；而 gRPC 直接基于 HTTP/2，Dubbo 3 推出的 Triple 协议也兼容 gRPC。HTTP/2 的二进制分帧和多路复用允许在一个 TCP 连接上交错传输多个请求和响应，解决了 HTTP/1.1 在应用层的队头阻塞（TCP 层的队头阻塞仍然存在）；采用标准协议还便于跨语言调用，也更容易被网关、Service Mesh 等基础设施识别和治理。

下面以 Netty 为基础来实现一个简易的 RPC 框架。

# 2 Protocol

其实作为 RPC 的协议，只需要考虑一个问题，就是怎么把一次本地方法的调用，变成能够被网络传输的字节流。

## 2.1 请求体与响应体

首先要定义方法的调用和返回这两个对象实体：

```java
//请求体
public class RpcRequest {
    private String id; //调用编号
    private String className;// 类名
    private String methodName;// 方法名
    private Class<?>[] parameterTypes;// 参数类型
    private Object[] parameters;// 参数列表
}

//响应体
public class RpcResponse {
    private String requestId; //对应的调用编号
    private int code; //响应码
    private String msg; //异常信息
    private Object data; //调用结果
}
```

## 2.2 序列化与反序列化

确定了需要序列化的对象实体，就要确定序列化的协议，并实现序列化和反序列化两个方法：

```java
public interface Serialization {
    <T> byte[] serialize (T obj);
    <T> T deSerialize (byte[] data, Class<T> clz);
}
```

可选用的序列化的协议很多，比如：

- JDK 自带的序列化方法（不推荐，不利于之后的跨语言调用）
- JSON，可读性强，但是序列化速度慢、体积大
- Protobuf、Kryo、Hessian 等都是优秀的序列化框架，也可按需选择

为了简单和便于调试，我们选择 JSON 作为序列化协议，使用 Jackson 作为解析框架（Fastjson 1.x 曾多次曝出反序列化漏洞，官方仓库已归档并建议升级到 Fastjson2，新项目不建议再使用）：

```java
public class JacksonSerialization implements Serialization {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public <T> byte[] serialize (T obj) {
        try {
            return MAPPER.writeValueAsBytes(obj);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("serialize failed", e);
        }
    }

    @Override
    public <T> T deSerialize (byte[] data, Class<T> clz) {
        try {
            return MAPPER.readValue(data, clz);
        } catch (IOException e) {
            throw new IllegalStateException("deserialize failed", e);
        }
    }
}
```

需要注意，JSON 不携带 Java 类型信息，`RpcRequest.parameters`、`RpcResponse.data` 这类 Object 字段中的自定义对象，反序列化后会变成 Map，服务端还需要结合 `parameterTypes` 再做一次类型转换。本文的示例只传递字符串等简单类型，所以没有处理这一点。

## 2.3 编码与解码

TCP 没有消息保护边界，需要在消息接收端处理消息边界问题，也就是我们所说的**粘包**、**拆包**问题，可以通过自定义通信协议的方式来解决。

通信协议就是通信双方约定好的数据格式，发送方按照这个数据格式来发送，接收方按照这个格式来解析。典型的协议包括：定长协议、特殊字符分隔符协议、报文头指定 Length 等。在确定了使用什么通信协议的情况下，发送方和接收方要完成的工作不同：

- **编码**：发送方要将待发送的数据转换成协议规定格式的二进制数据流，称之为**编码**（encode），编码功能由编码器（encoder）完成。
- **解码**：接收方需要根据协议的格式，对二进制数据进行解析，称之为**解码**（decode），解码功能由解码器（decoder）完成。
- **编解码**：如果有一种组件，既能编码，又能解码，则称之为**编码解码器**（codec）。这种组件在发送方和接收方都可以使用。

Netty 提供了一套完善的编解码框架，不论是公有协议还是私有协议，我们都可以在这个框架的基础上，非常容易地实现相应的编码器/解码器。输入的数据是在 `ChannelInboundHandler` 中处理的，输出的数据是在 `ChannelOutboundHandler` 中处理的。因此编码器/解码器实际上是这两个接口的特殊实现类，只不过它们的作用仅仅是编码/解码。

### 2.3.1 解码器

对于解码器，Netty 中主要提供了抽象基类 `ByteToMessageDecoder` 和 `MessageToMessageDecoder`。

![](imgs/2.png)

通常，ByteToMessageDecoder 解码后会得到一个 ByteBuf 实例列表，每个 ByteBuf 实例都包含了一个完整的报文信息。可以把这些 ByteBuf 实例直接交给之后的 ChannelInboundHandler 处理，或者将这些包含了完整报文信息的 ByteBuf 实例解析封装到不同的 Java 对象实例后，再交给其处理。不管哪一种情况，之后的 ChannelInboundHandler 在处理时都不需要再考虑粘包、拆包问题。

Netty 提供了一些 ByteToMessageDecoder 的常见实现类：

- **FixedLengthFrameDecoder**：定长协议解码器，我们可以指定固定的字节数算一个完整的报文
- **LineBasedFrameDecoder**：行分隔符解码器，遇到 `\n` 或者 `\r\n`，则认为是一个完整的报文
- **DelimiterBasedFrameDecoder**：分隔符解码器，与 LineBasedFrameDecoder 类似，只不过分隔符可以自己指定
- **LengthFieldBasedFrameDecoder**：基于长度字段的解码器，将报文划分为报文头/报文体，根据报文头中的 Length 字段确定报文体的长度，因此报文体的长度是可变的
- **JsonObjectDecoder**：JSON 格式解码器，当检测到匹配数量的 `{`、`}` 或 `[`、`]` 时，则认为是一个完整的 JSON 对象或者 JSON 数组

这些实现类都只是将接收到的二进制数据解码成包含完整报文信息的 ByteBuf 实例后，就直接交给了之后的 ChannelInboundHandler 处理。之所以不将 ByteBuf 中的信息封装到 Java 对象中，道理很简单：Netty 根本不知道开发者想封装到什么对象中，甚至不知道报文中的具体内容是什么，因此不如直接把包含了完整报文信息的 ByteBuf 实例交给开发人员自己来解析封装。

而 `MessageToMessageDecoder` 则是将一个本身就包含完整报文信息的对象转换成另一个 Java 对象。一个比较容易理解的类比是 Java Web 编程：通常客户端浏览器发送过来的二进制数据，已经被 Web 容器（如 Tomcat）解析成了一个 HttpServletRequest 对象，但是我们还是需要将 HttpServletRequest 中的数据提取出来，封装成我们自己的 POJO 类，也就是从一个 Java 对象（HttpServletRequest）转换成另一个 Java 对象（我们的 POJO 类）。

### 2.3.2 编码器

与 ByteToMessageDecoder 和 MessageToMessageDecoder 相对应，Netty 提供了对应的编码器实现 `MessageToByteEncoder` 和 `MessageToMessageEncoder`，二者都实现了 `ChannelOutboundHandler` 接口。

![](imgs/3.png)

相对来说，编码器比解码器的实现要更加简单，原因在于解码器除了要按照协议解析数据，还要处理粘包、拆包问题；而编码器只要将数据转换成协议规定的二进制格式发送即可。

### 2.3.3 编码解码器

编码解码器同时具有编码与解码功能，其特点是同时实现了 ChannelInboundHandler 和 ChannelOutboundHandler 接口，因此在数据输入和输出时都能进行处理。Netty 提供了一个 ChannelDuplexHandler 适配器类，编码解码器的抽象基类 `ByteToMessageCodec`、`MessageToMessageCodec` 都继承自此类，如下：

![](imgs/4.png)

# 3 Server

Netty 作为高性能的 NIO 通信框架，在很多 RPC 框架中都有它的身影。我们也用它作为通信服务器。

```java
public class RpcServer implements ApplicationContextAware, InitializingBean, DisposableBean {

    private static final Logger logger = LoggerFactory.getLogger(RpcServer.class);

    //注册表，用于存储暴露出去的RPC服务
    private RpcProviderRegistry rpcServiceRegistry = new RpcProviderRegistry();

    EventLoopGroup bossGroup = null;
    EventLoopGroup workGroup = null;

    private String host;
    private int port;

    public RpcServer(String host, int port) {
        this.host = host;
        this.port = port;
    }

    @Override
    public void afterPropertiesSet () throws Exception {
        start();
    }

    @Override
    public void setApplicationContext (ApplicationContext applicationContext) throws BeansException {
        Map<String, Object> beanMap = applicationContext.getBeansWithAnnotation(RpcService.class);
        beanMap.forEach((beanName, bean) -> {
            Class<?>[] interfaces = bean.getClass().getInterfaces();
            String interfaceName = interfaces[0].getName();
            logger.info("find rpc service: " + interfaceName);

            //被@RpcService标注的bean，会被自动注册为RPC服务
            rpcServiceRegistry.registerServiceBean(interfaceName, bean);
        });
    }

    /**
     * 启动服务器
    */
    private void start () {

        //Netty 是基于 Reactor 模型的，所以需要初始化两组线程 boss 和 worker
        // boss 负责分发请求，worker 负责执行相应的 handler
        bossGroup = new NioEventLoopGroup(1);
        workGroup = new NioEventLoopGroup(4);

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new RpcServerChannelInitializer(rpcServiceRegistry));//注册编解码器、服务处理器等

        try {
            bootstrap.bind(host, port).sync();
            logger.info("rpc server started");
        } catch (Exception e) {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
            throw new IllegalStateException("rpc server start failed", e);
        }

    }

    @Override
    public void destroy () {
        //Spring 容器关闭时释放线程资源
        bossGroup.shutdownGracefully();
        workGroup.shutdownGracefully();
    }

}
```

为了实现服务提供者的自动注册，这里利用了 Spring 容器的扩展点：`RpcServer` 实现了 `ApplicationContextAware` 接口，在 Spring 容器启动时，会扫描出被自定义注解 `@RpcService` 标记的 bean，放到 `RpcProviderRegistry` 中。它其实就相当于一个“接口名 → bean 实例”的 Map，便于 RpcServerHandler 获取 RPC 服务。

另外要注意，`start()` 运行在 Spring 的初始化流程中，只等待端口绑定完成即可，不能像独立的 Netty 程序那样调用 `channel().closeFuture().sync()`，否则会一直阻塞在这里，导致 Spring 容器无法完成启动。Netty 的 EventLoop 线程是非守护线程，绑定成功后进程不会退出。

`RpcServerChannelInitializer` 用于初始化 `ChannelPipeline`，注册服务端的编码器、解码器，以及服务处理的 Handler：

```java
public class RpcServerChannelInitializer extends ChannelInitializer<SocketChannel> {

    private RpcProviderRegistry rpcServiceRegistry;

    public RpcServerChannelInitializer (RpcProviderRegistry rpcServiceRegistry) {
        this.rpcServiceRegistry = rpcServiceRegistry;
    }

    @Override
    protected void initChannel (SocketChannel ch) throws Exception {

        ChannelPipeline pipeline = ch.pipeline();

        //InboundHandler，执行顺序为注册顺序
        pipeline.addLast(new LengthFieldBasedFrameDecoder(65535,0,4));
        pipeline.addLast(new CommonRpcDecoder(new JacksonSerialization(), RpcRequest.class));

        //OutboundHandler，执行顺序为注册顺序的逆序
        pipeline.addLast(new CommonRpcEncoder(new JacksonSerialization(), RpcResponse.class));
        pipeline.addLast(new RpcServerHandler(rpcServiceRegistry));

    }
```

我们选择 `LengthFieldBasedFrameDecoder` 来处理粘包、拆包问题，也就是说，客户端写入的请求报文要遵循“报文长度 + 报文体”的数据格式。

`RpcServerHandler` 继承自 `ChannelInboundHandlerAdapter`，有数据流入时会触发 `channelRead` 方法，数据处理是通过 Java 的反射实现的：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

  if (!(msg instanceof RpcRequest)) {
    //不是RPC请求，交给后续Handler处理
    ctx.fireChannelRead(msg);
    return;
  }

  //收到了RPC请求
  RpcRequest request = (RpcRequest)msg;
  logger.info("receive rpc request: {}", request.getId());
  RpcResponse response = new RpcResponse();
  response.setRequestId(request.getId());

  try {
    Object data = handle(request); //处理数据
    response.setData(data);
  } catch (Exception e) {
    logger.error("handle rpc request failed: {}", request.getId(), e);
    response.setMsg(e.getMessage());
    response.setCode(-1);
  }

  //调用结果写入，RpcRequest到此处理完毕，不再向后传播
  ctx.writeAndFlush(response);
}

private Object handle (RpcRequest request) throws Exception {

  //从注册表获取对应的服务
  String interfaceName = request.getClassName();
  Object serviceBean = rpcServiceRegistry.getServiceBean(interfaceName);
  if (serviceBean == null) {
    throw new Exception("no service provider: " + interfaceName);
  }

  Class<?> clazz = serviceBean.getClass();
  String methodName = request.getMethodName();
  Class<?>[] parameterTypes = request.getParameterTypes();
  Object[] parameters = request.getParameters();

  //利用反射，触发方法调用
  Method method = clazz.getMethod(methodName, parameterTypes);
  method.setAccessible(true);
  return method.invoke(serviceBean, parameters);
}
```

# 4 Client

与 Server 端类似，Client 也需要借助 Netty 来实现网络通信，只不过比 Server 端要简单些：

```java
public class RpcClient {

    private static final Logger logger = LoggerFactory.getLogger(RpcClient.class);

    private InetSocketAddress remoteAddress; //服务器地址

    public RpcClient(InetSocketAddress remoteAddress) {
        this.remoteAddress = remoteAddress;
    }

    private volatile Channel channel; //连接服务器成功后，会获取一个Channel
    private RpcResponsePool rpcResponsePool; //存储RPC的调用结果

    private void connect () {

        EventLoopGroup loopGroup = new NioEventLoopGroup();
        rpcResponsePool = new RpcResponsePool();

        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(loopGroup).
                channel(NioSocketChannel.class)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(new RpcClientChannelInitializer(rpcResponsePool)); //注册编解码器、客户端处理器等

        try {
            //连接服务器
            channel = bootstrap.connect(remoteAddress).sync().channel();
            logger.info("connect rpc server success");
        } catch (Exception e) {
            loopGroup.shutdownGracefully();
            throw new IllegalStateException("connect rpc server failed: " + remoteAddress, e);
        }
    }

    public RpcResponse sendRequest (RpcRequest request) throws InterruptedException {
        if (channel == null) {
            synchronized (this) {
                if (channel == null) { //双重检查，保证并发调用时只建立一次连接，且连接建立完成后才往下走
                    connect();
                }
            }
        }

        channel.writeAndFlush(request).sync(); //发送完成之前，会阻塞住

        return rpcResponsePool.takeResponse(request.getId());
    }

}
```

Netty 是一个异步框架，所有的返回都是基于 Future 和 Callback 机制。但是在上面的 `sendRequest` 方法中，将 RPC 的调用由异步转为了同步。也就是说，发起 RPC 调用后，调用方会阻塞在这里，直到服务端的处理结果返回。具体来说，有两个关键点：

- 第一是往 Channel 写入后，调用了 `sync()` 方法，在数据写入完成之前，线程会阻塞住
- 第二是依靠 `RpcResponsePool` 里面每个请求对应的阻塞队列 `ArrayBlockingQueue`（容量为 1），执行 take 的线程会阻塞，直到响应被放入队列为止

这里不能使用 `SynchronousQueue`：它的容量为 0，`offer()` 只有在恰好有线程阻塞在 `take()` 上时才会成功。如果服务端响应先于调用方执行 `takeResponse` 到达，`offer()` 会直接返回 false，响应被丢弃，调用方将永远阻塞。容量为 1 的队列可以先把响应暂存起来，无论响应和 take 谁先到都不会丢失。

```java
public class RpcResponsePool {

    //requestID ——》 调用结果
    private ConcurrentHashMap<String, BlockingQueue<RpcResponse>> queueMap = new ConcurrentHashMap<>();

    public void putRequest (String requestId) {
        //建立request的阻塞队列，容量为1，响应先到也能暂存
        BlockingQueue<RpcResponse> queue = new ArrayBlockingQueue<>(1);
        queueMap.put(requestId, queue);
    }

    public void putResponse (RpcResponse response) {
        queueMap.get(response.getRequestId()).offer(response);
    }

    public RpcResponse takeResponse (String requestId) {
        RpcResponse response = null;
        try {
            //在requestId对应的response放入之前，会阻塞在这里
            response =  queueMap.get(requestId).take();
            queueMap.remove(requestId);

        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return response;
    }
}
```

到这里可以解释一下，为什么要在 `RpcRequest` 和 `RpcResponse` 中增加一个 ID 了：因为 Netty 中的 Channel 是会被多个线程使用的，当一个结果异步地返回后，并不知道它对应的是哪个线程发出的请求。这个时候就需要利用一个 Map，建立 ID 和 `RpcResponse` 的映射，这样请求的线程只要使用对应的 ID，就能获取相应的返回结果。

客户端同样也需要注册编解码器与收发数据的处理器等：

```java
public class RpcClientChannelInitializer extends ChannelInitializer<SocketChannel> {

    private RpcResponsePool rpcResponsePool;

    public RpcClientChannelInitializer(RpcResponsePool rpcResponsePool) {
        this.rpcResponsePool = rpcResponsePool;
    }

    @Override
    protected void initChannel (SocketChannel ch) throws Exception {

        ChannelPipeline pipeline = ch.pipeline();

        pipeline.addLast(new LengthFieldBasedFrameDecoder(65535,0,4));
        pipeline.addLast(new CommonRpcEncoder(new JacksonSerialization(), RpcRequest.class));
        pipeline.addLast(new CommonRpcDecoder(new JacksonSerialization(), RpcResponse.class));

        pipeline.addLast(new RpcClientHandler(rpcResponsePool)); //既是InboundHandler，又是OutboundHandler，必须放在最后

    }

}
```

与 Server 端不同，`RpcClientHandler` 是 `ChannelDuplexHandler` 的子类，在数据写出时（发送 RpcRequest）触发 `write` 方法，在数据读入时（收到 RpcResponse）触发 `channelRead` 方法。

```java
public class RpcClientHandler extends ChannelDuplexHandler {

    private static final Logger logger = LoggerFactory.getLogger(RpcClientHandler.class);

    private RpcResponsePool rpcResponsePool;

    public RpcClientHandler(RpcResponsePool rpcResponsePool) {
        this.rpcResponsePool = rpcResponsePool;
    }

    @Override
    public void channelRead (ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof RpcResponse) {
            RpcResponse resp = (RpcResponse) msg;
            logger.info("receive server response: {}", resp.getRequestId());

            //接收到RPC结果，放到RpcResponsePool中，匹配发送时放入的请求ID
            rpcResponsePool.putResponse(resp);
        }
        super.channelRead(ctx, msg);
    }

    @Override
    public void write (ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        if (msg instanceof RpcRequest) {
            RpcRequest req = (RpcRequest) msg;
            logger.info("send request: {}", req.getId());

            //发送请求之前，先将请求ID放入RpcResponsePool
            rpcResponsePool.putRequest(req.getId());
        }
        super.write(ctx, msg, promise);
    }
}
```

至此，客户端与服务端的通信已经打通，但是还有一件非常重要的事情没有做：如何让客户端对 RPC 的调用无感知，也就是像调用本地方法那样调用远程服务？答案是利用动态代理，将 Netty 的通信细节屏蔽掉。

```java
public class RpcConsumerRegistry implements BeanFactoryPostProcessor {

    private List<RpcConsumerConfig> configList; //RPC服务的配置信息

    @Override
    public void postProcessBeanFactory (ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        if (CollectionUtils.isEmpty(configList)) {
            return;
        }

        //将代理类注册为Spring的bean
        configList.forEach(config -> {
            Object beanInstance = createProxy(config);
            configurableListableBeanFactory.registerSingleton(config.getBeanId(), beanInstance);
        });
    }

    /**
     * 创建远程服务的本地代理
    */
    public <T> T createProxy(RpcConsumerConfig config) throws BeansException{
        try {
            Class<?> clazz = Class.forName(config.getInterfaceClass());

            InetSocketAddress remoteAddress = InetSocketAddress.createUnresolved(config.getServerHost(), config.getServerPort());
            RpcClient rpcClient = new RpcClient(remoteAddress);

            RpcInvoker invoker = new RpcInvoker(rpcClient);
            return (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class<?>[]{clazz}, invoker);
        } catch (ClassNotFoundException e) {
            throw new BeanCreationException(e.getMessage());
        }
    }
}
```

`RpcConsumerConfig` 用于配置远程服务的信息，每个服务都会生成一个代理类，并注册到 Spring 容器当中。这里使用的是 JDK 的动态代理，它要求被代理的类型必须是接口，这对 RPC 来说正好合适，因为调用方手里只有服务的 API 接口。成熟的框架在代理方式上会有不同的取舍：Dubbo 默认使用 Javassist 在运行期生成代理类，以减少反射调用的开销；gRPC 则不使用动态代理，而是在编译期根据 proto 文件生成客户端 Stub。

使用 JDK 动态代理时，还需要一个 `InvocationHandler` 来处理真正的业务逻辑：

```java
public class RpcInvoker implements InvocationHandler {

    private static final Logger logger = LoggerFactory.getLogger(RpcInvoker.class);

    private RpcClient rpcClient;

    public RpcInvoker (RpcClient rpcClient) {
        this.rpcClient = rpcClient;
    }

    @Override
    public Object invoke (Object proxy, Method method, Object[] args) throws Throwable {

        RpcRequest r = new RpcRequest();
        r.setId(UUID.randomUUID().toString());
        r.setClassName(method.getDeclaringClass().getName());
        r.setMethodName(method.getName());
        r.setParameterTypes(method.getParameterTypes());
        r.setParameters(args);

        //通过Netty发送请求，并接收调用结果
        RpcResponse response = rpcClient.sendRequest(r);
        if (response.getCode() == -1) {
            throw new RuntimeException("invoke rpc service failed: " + response.getMsg());
        }

        return response.getData();
    }

}
```

`invoke` 方法主要做了三件事：

- 生成 RequestId
- 拼装 RpcRequest
- 调用 RpcClient 发送请求，获取结果

# 5 Test

假设服务端定义了一个查询当前时间的接口：

```java
public interface TimeService {
    String getTime(); //返回服务器的当前时间
}
```

并将其实现，暴露出去：

```java
@RpcService //被标记为服务提供者
@Component
public class TimeServiceImpl implements TimeService {

    private static final String TIME_PATTERN = "yyyy-MM-dd HH:mm:ss";

    @Override
    public String getTime () {
        LocalDateTime now = LocalDateTime.now();
        String timeResp = now.format(DateTimeFormatter.ofPattern(TIME_PATTERN));
        return timeResp;
    }

}
```

服务端配置好 IP 与端口号，启动即可：

```java
@Configuration
public class RpcServerConfig {

    @Value("${rpc.server.host}")
    private String host;

    @Value("${rpc.server.port}")
    private int port;

    @Bean
    public RpcServer rpcServer() {
        return new RpcServer(host, port);
    }
}

@SpringBootApplication
public class ServerApp {

    public static void main(String [] args) {
        ConfigurableApplicationContext context = SpringApplication.run(ServerApp.class);
    }
}
```

服务端启动后，会监听指定端口，等待请求。

客户端依赖服务端的 API 包，在自己的类当中引入 `TimeService`：

```java
@Component
public class TimeEchoService {

    @Autowired
    private TimeService timeService; //直接注入RPC服务

    public void printBeiJingTime () {
        String time = timeService.getTime();
        System.out.println("当前北京时间为：" + time);
    }
}
```

客户端的 jar 包中并没有 `TimeService` 的实现类，但是通过动态代理我们模拟了一个实现类，并将其注册到 Spring 容器当中，如此一来 `TimeService` 就可以被当作本地服务一样，使用 `@Autowired` 等注解来注入和管理。当发生真正的调用时，代理类会与远程服务交互，完成字节传输、序列化、反序列化、对象转换等工作，达到上层无感知的效果。

客户端的配置与启动方式如下：

```java
@Configuration
public class RpcClientConfig {

    @Bean
    public List<RpcConsumerConfig> rpcConsumerConfigList() {

        List<RpcConsumerConfig> configList = Lists.newArrayList();

        //服务提供者的配置信息
        RpcConsumerConfig timeServiceConfig = new RpcConsumerConfig();
        timeServiceConfig.setServerHost("127.0.0.1");
        timeServiceConfig.setServerPort(13888);
        timeServiceConfig.setInterfaceClass("clf.winner.netty.rpc.test.server.api.TimeService");
        timeServiceConfig.setBeanId("timeService");

        configList.add(timeServiceConfig);

        return configList;
    }

    @Bean
    public static RpcConsumerRegistry rpcConsumerRegistry(List<RpcConsumerConfig> rpcConsumerConfigList) {
        RpcConsumerRegistry registry = new RpcConsumerRegistry();
        registry.setConfigList(rpcConsumerConfigList);

        return registry;
    }

}

@SpringBootApplication
public class ClientApp {

    public static void main(String [] args) {

        ConfigurableApplicationContext context = SpringApplication.run(ClientApp.class);

        TimeEchoService timeEchoService = (TimeEchoService)context.getBean("timeEchoService");

        timeEchoService.printBeiJingTime();

    }

}
```

完整代码的 GitHub 地址：[sample-netty-rpc](https://github.com/Winner2015/sample-netty-rpc)

当然，以上代码只包含最基础的远程调用功能，实际上一个完整的 RPC 框架还需要很多其他功能，比如服务的自动注册与发现（可借助 ZooKeeper、Nacos、Consul 等注册中心，部署在 Kubernetes 上时也可以直接使用 Service 机制）、负载均衡、心跳检测、超时与重试、服务治理等。
