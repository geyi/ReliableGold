Netty源码解析：https://blog.csdn.net/weixin_41385912/article/details/110944462

# Netty服务端启动过程
bossGroup 用于接收 TCP 连接请求，建立连接后会把连接转接给 workerGroup 来处理连接上的后续请求，比如 **读取请求数据-->解码请求数据-->进行业务处理-->编码响应数据-->发送响应数据** 这一整套流程。

EventLoopGroup 是一个线程组，其中的每一个线程都在循环执行着三件事情：
- select：轮训注册在其中的 Selector 上的 Channel 的 IO 事件
- processSelectedKeys：在对应的 Channel 上处理 IO 事件
- runAllTasks：再去以此循环处理任务队列中的其他任务

## `EventLoopGroup workerGroup = new NioEventLoopGroup()`的执行源码总结如下：
1. NioEventLoopGroup 的无参数构造函数会调用 NioEventLoopGroup 的有参数构造函数，最终把参数
    ```
    nThreads = 16
    executor = null
    selectorProvider = SelectorProvider.provider()
    selectStrategyFactory = DefaultSelectStrategyFactory.INSTANCE
    rejectedExecutionHandler = RejectedExecutionHandlers.reject()
    chooserFactory = DefaultEventExecutorChooserFactory.INSTANCE
    ```
    传递给父类 MultithreadEventLoopGroup 的有参数构造函数。
2. 父类 MultithreadEventLoopGroup 的有参数构造函数创建一个 NioEventLoop 的数组 `children = new EventExecutor[nThreads];`，并构建出 16 个 NioEventLoop 的实例放入其中。
3. 构建每一个 NioEventLoop 调用的是 `children[i] = newChild(executor, args);`。
4. newChild()方法最终调用了 NioEventLoop 的构造函数，初始化其中的选择器、任务队列、执行器等变量。

## ServerBootstrap 实例的创建与配置源码总结如下：
1. `group(bossGroup, workerGroup)`把 bossGroup 和 workerGroup 两个参数赋值给 ServerBootstrap 的成员变量 group（从父类 AbstractBootstrap 继承而来）和 childGroup。
2. `channel(NioServerSocketChannel.class)`通过反射机制给当前 ServerBootstrap 中的 channelFactory 属性（从父类 AbstractBootstrap 继承而来）赋值。在调用.bind()的时候 channelFactory 会创建 NioServerSocketChannel 的实例（同时创建了 Pipeline 和 NioServerSocketChannelConfig，并初始化了SIZE_TABLE）。
3. `option(ChannelOption.SO_BACKLOG, 100)`将可选项放入一个 ChannelOption 集合中。
4. `handler(new LoggingHandler(LogLevel.INFO))`将 Netty 提供的一个日志记录 Handler 赋值给 ServerBootstrap 实例的 handler 属性（从父类 AbstractBootstrap 继承而来）。这个 Handler 最终在 ServerBootstrap.init()方法中被放入 NioServerSocketChannel 实例的 pipeline 中。
5. `childHandler(...)`的作用是为接收客户端连接请求产生的 NioSocketChannel 实例的 pipeline 添加 Handler，例如：SSL 加密处理的 Handler，以及一个我们自定义的 Handler。

## ServerBootstrap 实例的.bind(PORT)源码总结如下：
1. 首先调用 AbstractBootstrap 中的 doBind() 方法完成 NioServerSocketChannel 实例的初始化和注册。
2. 然后调用 NioServerSocketChannel 实例的 bind() 方法。
3. NioServerSocketChannel 实例的 bind() 方法最终调用 sun.nio.ch.Net 中的 bind() 和 listen() 完成端口绑定和客户端连接监听。
4. sun.nio.ch.Net 中的 bind() 和 listen() 底层都是 JVM 进行的系统调用。
5. bind 完成后会进入 NioEventLoop 中的死循环，不断执行以下三个过程
   - select：轮训注册在其中的 Selector 上的 Channel 的 IO 事件
   - processSelectedKeys：在对应的 Channel 上处理 IO 事件
   - runAllTasks：再去以此循环处理任务队列中的其他任务

服务器端的 NioServerSocketChannel 实例将自己注册到 bossGroup 中 EventLoop 的 Selector 上。（注册的代码在`initAndRegister()`方法的`ChannelFuture regFuture = config().group().register(channel);`）

Netty 服务端接收客户端连接请求的总体流程为：监听 Accept 事件，接受连接-->创建一个新的 NioSocketChannel-->将新的 NioSocketChannel 注册到 workerGroup 上-->监听 NioSocketChannel 上的 I/O 事件。

# Netty 服务端接收客户端连接请求
## Netty 服务端接收客户端连接请求的源码总结如下：
1. 服务器端 bossGroup 中的 EventLoop 轮训 Accept 事件、获取事件后在 processSelectedKey()方法中调用 unsafe.read()方法，这个 unsafe 是内部类 io.netty.channel.nio.AbstractNioChannel.NioUnsafe 的实例，unsafe.read()方法由两个核心步骤组成：doReadMessages()和 pipeline.fireChannelRead()。
2. doReadMessages() 用于创建 NioSocketChannel 对象，包装了 JDK 的 SocketChannel 对象，并且添加了 pipeline、unsafe、config 等变量。
3. pipeline.fireChannelRead() 用于触发服务端 NioServerSocketChannel 的所有入站 Handler 的 channelRead()方法，在其中的一个类型为 ServerBootstrapAcceptor 的入站 Handler 的 channelRead() 方法中将新创建的 NioSocketChannel 对象注册到 workerGroup 中的一个 EventLoop 上，该 EventLoop 开始监听 NioSocketChannel 中的读事件。

# Channel、ChannelPipeline、ChannelHandler、ChannelHandlerContext
1. 每当 ServerSocketChannel 创建一个新的连接，就会创建一个 SocketChannel 对应目标客户端；
2. 每个 Netty Channel 包含了一个 ChannelPipeline（其实 Channel 和 ChannelPipeline 互相引用）；
3. 每个 ChannelPipeline 又维护了一个由 ChannelHandlerContext 构成的双向循环列表；
4. 其中的每一个 ChannelHandlerContext 都包含一个 ChannelHandler。

## ChannelPipeline
事件会依次流经 Pipeline 中每一个 ChannelHandlerContext，OutboundHandler 处理出站事件；InboundHandler 处理入站事件。

## ChannelHandler
通常，一个 Pipeline 中有多个 Handler，例如一个典型的服务器在每个管道中都会有协议解码器、协议编码器、业务处理程序，分别用来将接收到的二进制数据转换为 Java 对象，以及将要发送的 Java 转换为二进制数据，以及根据接收到的 Java 对象执行业务处理过程。

> 如果 Handler 里面的数据处理过程很快，可以放在当前 EventLoop 中处理，否则就要放入任务队列进行异步处理，或者开辟新的线程来处理。

## ChannelHandlerContext
ChannelHandlerContext 是一个接口，并且继承了 AttributeMap、ChannelInboundInvoker、ChannelOutboundInvoker（所谓 Invoker 就是触发器/调用器，用来调用 ChannelInboundHandler 或者 ChannelOutboundHandler 中的事件处理方法）三个接口。

## ChannelPipeline、ChannelHandler、ChannelHandlerContext 三者的创建过程
1. 任何一个 ChannelSocket（无论是 NioSocketChannel 还是 NioServerSocketChannel）创建的时候都会创建一个 ChannelPipeline，这发生在调用 AbstractChannel（它是 NioSocketChannel 和 NioServerSocketChannel 的父类）的构造方法的时候。
2. 当系统内部或者使用者调用 ChannelPipeline 的 addxxx 方法添加 new 出来的 ChannelHandler 实例的时候，都会创建一个包装该 ChannelHandler 的 ChannelHandlerContext。
3. 这些 ChannelHandlerContext 构成了 ChannelPipeline 中的双向循环链表。