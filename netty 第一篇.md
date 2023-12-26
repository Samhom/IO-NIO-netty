# netty 第一篇

## 1 **Netty 概述**

Netty 的设计主要基于主从 Reactor 多线程模式，并做了一定的改进。本节将使用一种渐进式的描述方式展示 Netty 的模样，即先给出 Netty 的简单版本，然后逐渐丰富其细节，直至展示出 Netty 的全貌。

简单版本的 Netty 的模样如下：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/4csj6uyvx8.png)

关于这张图，作以下几点说明：

1）BossGroup 线程维护 Selector，ServerSocketChannel 注册到这个 Selector 上，只关注连接建立请求事件（相当于主 Reactor）。

2）当接收到来自客户端的连接建立请求事件的时候，通过 ServerSocketChannel.accept 方法获得对应的 SocketChannel，并封装成 NioSocketChannel 注册到 WorkerGroup 线程中的 Selector，每个 Selector 运行在一个线程中（相当于从 Reactor）。

3）当 WorkerGroup 线程中的 Selector 监听到自己感兴趣的 IO 事件后，就调用 Handler 进行处理。

我们给这简单版的 Netty 添加一些细节：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/uimeso37m4.png)

关于这张图，作以下几点说明：

1）有两组线程池：BossGroup 和 WorkerGroup，BossGroup 中的线程（可以有多个，图中只画了一个）专门负责和客户端建立连接，WorkerGroup 中的线程专门负责处理连接上的读写。

2）BossGroup 和 WorkerGroup 含有多个不断循环的执行事件处理的线程，每个线程都包含一个 Selector，用于监听注册在其上的 Channel。

3）每个 BossGroup 中的线程循环执行以下三个步骤：

3.1）轮训注册在其上的 ServerSocketChannel 的 accept 事件（OP_ACCEPT 事件）

3.2）处理 accept 事件，与客户端建立连接，生成一个 NioSocketChannel，并将其注册到 WorkerGroup 中某个线程上的 Selector 上

3.3）再去以此循环处理任务队列中的下一个事件

4）每个 WorkerGroup 中的线程循环执行以下三个步骤：

4.1）轮训注册在其上的 NioSocketChannel 的 read/write 事件（OP_READ/OP_WRITE 事件）

4.2）在对应的 NioSocketChannel 上处理 read/write 事件

4.3）再去以此循环处理任务队列中的下一个事件

我们再来看下终极版的 Netty 的模样，如下图所示（图片来源于网络）：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/ocmrm2pw9j.png)

关于这张图，作以下几点说明：

1）Netty 抽象出两组线程池：BossGroup 和 WorkerGroup，也可以叫做 BossNioEventLoopGroup 和 WorkerNioEventLoopGroup。每个线程池中都有 NioEventLoop 线程。BossGroup 中的线程专门负责和客户端建立连接，WorkerGroup 中的线程专门负责处理连接上的读写。BossGroup 和 WorkerGroup 的类型都是 NioEventLoopGroup。

2）NioEventLoopGroup 相当于一个事件循环组，这个组中含有多个事件循环，每个事件循环就是一个 NioEventLoop。

3）NioEventLoop 表示一个不断循环的执行事件处理的线程，每个 NioEventLoop 都包含一个 Selector，用于监听注册在其上的 Socket 网络连接（Channel）。

4）NioEventLoopGroup 可以含有多个线程，即可以含有多个 NioEventLoop。

5）每个 BossNioEventLoop 中循环执行以下三个步骤：

5.1）**select**：轮训注册在其上的 ServerSocketChannel 的 accept 事件（OP_ACCEPT 事件）

5.2）**processSelectedKeys**：处理 accept 事件，与客户端建立连接，生成一个 NioSocketChannel，并将其注册到某个 WorkerNioEventLoop 上的 Selector 上

5.3）**runAllTasks**：再去以此循环处理任务队列中的其他任务

6）每个 WorkerNioEventLoop 中循环执行以下三个步骤：

6.1）**select**：轮训注册在其上的 NioSocketChannel 的 read/write 事件（OP_READ/OP_WRITE 事件）

6.2）**processSelectedKeys**：在对应的 NioSocketChannel 上处理 read/write 事件

6.3）**runAllTasks**：再去以此循环处理任务队列中的其他任务

7）在以上两个**processSelectedKeys**步骤中，会使用 Pipeline（管道），Pipeline 中引用了 Channel，即通过 Pipeline 可以获取到对应的 Channel，Pipeline 中维护了很多的处理器（拦截处理器、过滤处理器、自定义处理器等）。这里暂时不详细展开讲解 Pipeline。

**1 基于 Netty 的 TCP Server/Client 案例**

下面我们写点代码来加深理解 Netty 的模样。下面两段代码分别是基于 Netty 的 TCP Server 和 TCP Client。

服务端代码为：

```javascript
/**
 * 需要的依赖：
 * <dependency>
 * <groupId>io.netty</groupId>
 * <artifactId>netty-all</artifactId>
 * <version>4.1.52.Final</version>
 * </dependency>
 */
public static void main(String[] args) throws InterruptedException {

    // 创建 BossGroup 和 WorkerGroup
    // 1. bossGroup 只处理连接请求
    // 2. 业务处理由 workerGroup 来完成
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
        // 创建服务器端的启动对象
        ServerBootstrap bootstrap = new ServerBootstrap();
        // 配置参数
        bootstrap
                // 设置线程组
                .group(bossGroup, workerGroup)
                // 说明服务器端通道的实现类（便于 Netty 做反射处理）
                .channel(NioServerSocketChannel.class)
                // 设置等待连接的队列的容量（当客户端连接请求速率大
             // 于 NioServerSocketChannel 接收速率的时候，会使用
                // 该队列做缓冲）
                // option()方法用于给服务端的 ServerSocketChannel
                // 添加配置
                .option(ChannelOption.SO_BACKLOG, 128)
                // 设置连接保活
                // childOption()方法用于给服务端 ServerSocketChannel
                // 接收到的 SocketChannel 添加配置
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                // handler()方法用于给 BossGroup 设置业务处理器
                // childHandler()方法用于给 WorkerGroup 设置业务处理器
                .childHandler(
                        // 创建一个通道初始化对象
                        new ChannelInitializer<SocketChannel>() {
                            // 向 Pipeline 添加业务处理器
                            @Override
                            protected void initChannel(
                                    SocketChannel socketChannel
                            ) throws Exception {
                                socketChannel.pipeline().addLast(
                                        new NettyServerHandler()
                                );
                                
                                // 可以继续调用 socketChannel.pipeline().addLast()
                                // 添加更多 Handler
                            }
                        }
                );

        System.out.println("server is ready...");

        // 绑定端口，启动服务器，生成一个 channelFuture 对象，
        // ChannelFuture 涉及到 Netty 的异步模型，后面展开讲
        ChannelFuture channelFuture = bootstrap.bind(8080).sync();
        // 对通道关闭进行监听
        channelFuture.channel().closeFuture().sync();
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}

/**
 * 自定义一个 Handler，需要继承 Netty 规定好的某个 HandlerAdapter（规范）
 * InboundHandler 用于处理数据流入本端（服务端）的 IO 事件
 * InboundHandler 用于处理数据流出本端（服务端）的 IO 事件
 */
static class NettyServerHandler extends ChannelInboundHandlerAdapter {
    /**
     * 当通道有数据可读时执行
     *
     * @param ctx 上下文对象，可以从中取得相关联的 Pipeline、Channel、客户端地址等
     * @param msg 客户端发送的数据
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        // 接收客户端发来的数据

        System.out.println("client address: "
                + ctx.channel().remoteAddress());

        // ByteBuf 是 Netty 提供的类，比 NIO 的 ByteBuffer 性能更高
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("data from client: "
                + byteBuf.toString(CharsetUtil.UTF_8));
    }

    /**
     * 数据读取完毕后执行
     *
     * @param ctx 上下文对象
     * @throws Exception
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx)
            throws Exception {
        // 发送响应给客户端
        ctx.writeAndFlush(
                // Unpooled 类是 Netty 提供的专门操作缓冲区的工具
                // 类，copiedBuffer 方法返回的 ByteBuf 对象类似于
                // NIO 中的 ByteBuffer，但性能更高
                Unpooled.copiedBuffer(
                        "hello client! i have got your data.",
                        CharsetUtil.UTF_8
                )
        );
    }

    /**
     * 发生异常时执行
     *
     * @param ctx   上下文对象
     * @param cause 异常对象
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        // 关闭与客户端的 Socket 连接
        ctx.channel().close();
    }
}
```

客户端端代码为：

```javascript
/**
 * 需要的依赖：
 * <dependency>
 * <groupId>io.netty</groupId>
 * <artifactId>netty-all</artifactId>
 * <version>4.1.52.Final</version>
 * </dependency>
 */
public static void main(String[] args) throws InterruptedException {

    // 客户端只需要一个事件循环组，可以看做 BossGroup
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

    try {
        // 创建客户端的启动对象
        Bootstrap bootstrap = new Bootstrap();
        // 配置参数
        bootstrap
                // 设置线程组
                .group(eventLoopGroup)
                // 说明客户端通道的实现类（便于 Netty 做反射处理）
                .channel(NioSocketChannel.class)
                // handler()方法用于给 BossGroup 设置业务处理器
                .handler(
                        // 创建一个通道初始化对象
                        new ChannelInitializer<SocketChannel>() {
                            // 向 Pipeline 添加业务处理器
                            @Override
                            protected void initChannel(
                                    SocketChannel socketChannel
                            ) throws Exception {
                                socketChannel.pipeline().addLast(
                                        new NettyClientHandler()
                                );
                                
                                // 可以继续调用 socketChannel.pipeline().addLast()
                                // 添加更多 Handler
                            }
                        }
                );

        System.out.println("client is ready...");

        // 启动客户端去连接服务器端，ChannelFuture 涉及到 Netty 的异步模型，后面展开讲
        ChannelFuture channelFuture = bootstrap.connect(
                "127.0.0.1",
                8080).sync();
        // 对通道关闭进行监听
        channelFuture.channel().closeFuture().sync();
    } finally {
        eventLoopGroup.shutdownGracefully();
    }
}

/**
 * 自定义一个 Handler，需要继承 Netty 规定好的某个 HandlerAdapter（规范）
 * InboundHandler 用于处理数据流入本端（客户端）的 IO 事件
 * InboundHandler 用于处理数据流出本端（客户端）的 IO 事件
 */
static class NettyClientHandler extends ChannelInboundHandlerAdapter {
    /**
     * 通道就绪时执行
     *
     * @param ctx 上下文对象
     * @throws Exception
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx)
            throws Exception {
        // 向服务器发送数据
        ctx.writeAndFlush(
                // Unpooled 类是 Netty 提供的专门操作缓冲区的工具
                // 类，copiedBuffer 方法返回的 ByteBuf 对象类似于
                // NIO 中的 ByteBuffer，但性能更高
                Unpooled.copiedBuffer(
                        "hello server!",
                        CharsetUtil.UTF_8
                )
        );
    }

    /**
     * 当通道有数据可读时执行
     *
     * @param ctx 上下文对象
     * @param msg 服务器端发送的数据
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        // 接收服务器端发来的数据

        System.out.println("server address: "
                + ctx.channel().remoteAddress());

        // ByteBuf 是 Netty 提供的类，比 NIO 的 ByteBuffer 性能更高
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("data from server: "
                + byteBuf.toString(CharsetUtil.UTF_8));
    }

    /**
     * 发生异常时执行
     *
     * @param ctx   上下文对象
     * @param cause 异常对象
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        // 关闭与服务器端的 Socket 连接
        ctx.channel().close();
    }
}
```

什么？你觉得使用 Netty 编程难度和工作量更大了？不会吧不会吧，你要知道，你通过这么两段简短的代码得到了一个基于主从 Reactor 多线程模式的服务器，一个高吞吐量和并发量的服务器，一个异步处理服务器……你还要怎样？

对上面的两段代码，作以下简单说明：

1）Bootstrap 和 ServerBootstrap 分别是客户端和服务器端的引导类，一个 Netty 应用程序通常由一个引导类开始，主要是用来配置整个 Netty 程序、设置业务处理类（Handler）、绑定端口、发起连接等。

2）客户端创建一个 NioSocketChannel 作为客户端通道，去连接服务器。

3）服务端首先创建一个 NioServerSocketChannel 作为服务器端通道，每当接收一个客户端连接就产生一个 NioSocketChannel 应对该客户端。

4）使用 Channel 构建网络 IO 程序的时候，不同的协议、不同的阻塞类型和 Netty 中不同的 Channel 对应，常用的 Channel 有：

- NioSocketChannel：非阻塞的 TCP 客户端 Channel（本案例的客户端使用的 Channel）
- NioServerSocketChannel：非阻塞的 TCP 服务器端 Channel（本案例的服务器端使用的 Channel）
- NioDatagramChannel：非阻塞的 UDP Channel
- NioSctpChannel：非阻塞的 SCTP 客户端 Channel
- NioSctpServerChannel：非阻塞的 SCTP 服务器端 Channel ......

启动服务端和客户端代码，调试以上的服务端代码，发现：

1）默认情况下 BossGroup 和 WorkerGroup 都包含 16 个线程（NioEventLoop），这是因为我的 PC 是 8 核的 NioEventLoop 的数量=coreNum*2。这 16 个线程相当于主 Reactor。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/v32cgdplc3.png)

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/6hynq48ppt.png)

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/oh2kvunedz.png)

其实创建 BossGroup 和 WorkerGroup 的时候可以指定 NioEventLoop 数量，如下：

```javascript
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(16);
```

这样就能更好地分配线程资源。

2）每一个 NioEventLoop 包含如下的属性（比如自己的 Selector、任务队列、执行器等）：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/wk3jpoapcs.png)

3）将代码断在服务端的 NettyServerHandler.channelRead 上：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/6wo59e9vsc.png)

可以看到 ctx 中包含的属性如下：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/e16gputcqg.png)

可以看到：

- 当前 ChannelHandlerContext ctx 是位于 ChannelHandlerContext 责任链中的一环，可以看到其 next、prev 属性
- 当前 ChannelHandlerContext ctx 包含一个 Handler
- 当前 ChannelHandlerContext ctx 包含一个 Pipeline
- Pipeline 本质上是一个双向循环列表，可以看到其 tail、head 属性
- Pipeline 中包含一个 Channel，Channel 中又包含了该 Pipeline，两者互相引用 ……

从下一节开始，我将深入剖析以上两段代码，向读者展示 Netty 的更多细节。

## **2 Netty 的 Handler 组件**

无论是服务端代码中自定义的 NettyServerHandler 还是客户端代码中自定义的 NettyClientHandler，都继承于 ChannelInboundHandlerAdapter，ChannelInboundHandlerAdapter 又继承于 ChannelHandlerAdapter，ChannelHandlerAdapter 又实现了 ChannelHandler：

```javascript
public class ChannelInboundHandlerAdapter 
    extends ChannelHandlerAdapter 
    implements ChannelInboundHandler {
    ......
```

```javascript
public abstract class ChannelHandlerAdapter 
    implements ChannelHandler {
    ......
```

因此无论是服务端代码中自定义的 NettyServerHandler 还是客户端代码中自定义的 NettyClientHandler，都可以统称为 ChannelHandler。

Netty 中的 ChannelHandler 的作用是，在当前 ChannelHandler 中处理 IO 事件，并将其传递给 ChannelPipeline 中下一个 ChannelHandler 处理，因此多个 ChannelHandler 形成一个责任链，责任链位于 ChannelPipeline 中。

数据在基于 Netty 的服务器或客户端中的处理流程是：读取数据-->解码数据-->处理数据-->编码数据-->发送数据。其中的每个过程都用得到 ChannelHandler 责任链。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/1mwzt4el0l.png)

Netty 中的 ChannelHandler 体系如下（第一张图来源于网络）：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/4ro84blxwr.png)

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/1p8dok57zw.png)

其中：

- ChannelInboundHandler 用于处理入站 IO 事件
- ChannelOutboundHandler 用于处理出站 IO 事件
- ChannelInboundHandlerAdapter 用于处理入站 IO 事件
- ChannelOutboundHandlerAdapter 用于处理出站 IO 事件

ChannelPipeline 提供了 ChannelHandler 链的容器。以客户端应用程序为例，如果事件的方向是从客户端到服务器的，我们称事件是出站的，那么客户端发送给服务器的数据会通过 Pipeline 中的一系列 ChannelOutboundHandler 进行处理；如果事件的方向是从服务器到客户端的，我们称事件是入站的，那么服务器发送给客户端的数据会通过 Pipeline 中的一系列 ChannelInboundHandler 进行处理。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/t4tmv79kr1.png)

无论是服务端代码中自定义的 NettyServerHandler 还是客户端代码中自定义的 NettyClientHandler，都继承于 ChannelInboundHandlerAdapter，ChannelInboundHandlerAdapter 提供的方法如下：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/5nbgbkq2yo.png)

从方法名字可以看出，它们在不同的事件发生后被触发，例如注册 Channel 时执行 channelRegistred()、添加 ChannelHandler 时执行 handlerAdded()、收到入站数据时执行 channelRead()、入站数据读取完毕后执行 channelReadComplete()等等。

## **3 Netty 的 Pipeline 组件**

上一节说到，Netty 的 ChannelPipeline，它维护了一个 ChannelHandler 责任链，负责拦截或者处理 inbound（入站）和 outbound（出站）的事件和操作。这一节给出更深层次的描述。

ChannelPipeline 实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个 ChannelHandler 如何相互交互。

每个 Netty Channel 包含了一个 ChannelPipeline（其实 Channel 和 ChannelPipeline 互相引用），而 ChannelPipeline 又维护了一个由 ChannelHandlerContext 构成的双向循环列表，其中的每一个 ChannelHandlerContext 都包含一个 ChannelHandler。（前文描述的时候为了简便，直接说 ChannelPipeline 包含了一个 ChannelHandler 责任链，这里给出完整的细节。）

如下图所示（图片来源于网络）：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/wp7sk2fuyb.jpeg)

还记得下面这张图吗？这是上文中基于 Netty 的 Server 程序的调试截图，可以从中看到 ChannelHandlerContext 中包含了哪些成分：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/e16gputcqg.png)

ChannelHandlerContext 除了包含 ChannelHandler 之外，还关联了对应的 Channel 和 Pipeline。可以这么来讲：ChannelHandlerContext、ChannelHandler、Channel、ChannelPipeline 这几个组件之间互相引用，互为各自的属性，你中有我、我中有你。

在处理入站事件的时候，入站事件及数据会从 Pipeline 中的双向链表的头 ChannelHandlerContext 流向尾 ChannelHandlerContext，并依次在其中每个 ChannelInboundHandler（例如解码 Handler）中得到处理；出站事件及数据会从 Pipeline 中的双向链表的尾 ChannelHandlerContext 流向头 ChannelHandlerContext，并依次在其中每个 ChannelOutboundHandler（例如编码 Handler）中得到处理。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/g6abk7igyq.png)

## **4  Netty 的 EventLoopGroup 组件**

在基于 Netty 的 TCP Server 代码中，包含了两个 EventLoopGroup——bossGroup 和 workerGroup，EventLoopGroup 是一组 EventLoop 的抽象。

追踪 Netty 的 EventLoop 的继承链，可以发现 EventLoop 最终继承于 JUC Executor，因此 EventLoop 本质就是一个 JUC Executor，即线程，JUC Executor 的源码为：

```javascript
public interface Executor {
    /**
     * Executes the given command at some time in the future.
     */
    void execute(Runnable command);
}
```



Netty 为了更好地利用多核 CPU 的性能，一般会有多个 EventLoop 同时工作，每个 EventLoop 维护着一个 Selector 实例，Selector 实例监听注册其上的 Channel 的 IO 事件。

EventLoopGroup 含有一个 next 方法，它的作用是按照一定规则从 Group 中选取一个 EventLoop 处理 IO 事件。

在服务端，通常 Boss EventLoopGroup 只包含一个 Boss EventLoop（单线程），该 EventLoop 维护者一个注册了 ServerSocketChannel 的 Selector 实例。该 EventLoop 不断轮询 Selector 得到 OP_ACCEPT 事件（客户端连接事件），然后将接收到的 SocketChannel 交给 Worker EventLoopGroup，Worker EventLoopGroup 会通过 next()方法选取一个 Worker EventLoop 并将这个 SocketChannel 注册到其中的 Selector 上，由这个 Worker EventLoop 负责该 SocketChannel 上后续的 IO 事件处理。整个过程如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/puxdjxvdke.png)

## **5 Netty 的 TaskQueue**

在 Netty 的每一个 NioEventLoop 中都有一个 TaskQueue，设计它的目的是在任务提交的速度大于线程的处理速度的时候起到缓冲作用。或者用于异步地处理 Selector 监听到的 IO 事件。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/wfs17ovmju.png)

Netty 中的任务队列有三种使用场景：

1）处理用户程序的自定义普通任务的时候

2）处理用户程序的自定义定时任务的时候

3）非当前 Reactor 线程调用当前 Channel 的各种方法的时候。

对于第一种场景，举个例子，2.4 节的基于 Netty 编写的服务端的 Handler 中，假如 channelRead 方法中执行的过程很耗时，那么以下的阻塞式处理方式无疑会降低当前 NioEventLoop 的并发度：

```javascript
/**
 * 当通道有数据可读时执行
 *
 * @param ctx 上下文对象
 * @param msg 客户端发送的数据
 * @throws Exception
 */
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {
    // 借助休眠模拟耗时操作
    Thread.sleep(LONG_TIME);

    ByteBuf byteBuf = (ByteBuf) msg;
    System.out.println("data from client: "
            + byteBuf.toString(CharsetUtil.UTF_8));
}
```

改进方法就是借助任务队列，代码如下：

```javascript
/**
 * 当通道有数据可读时执行
 *
 * @param ctx 上下文对象
 * @param msg 客户端发送的数据
 * @throws Exception
 */
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {
    // 假如这里的处理非常耗时，那么就需要借助任务队列异步执行

    final Object finalMsg = msg;

    // 通过 ctx.channel().eventLoop().execute()将耗时
    // 操作放入任务队列异步执行
    ctx.channel().eventLoop().execute(new Runnable() {
        public void run() {
            // 借助休眠模拟耗时操作
            try {
                Thread.sleep(LONG_TIME);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            ByteBuf byteBuf = (ByteBuf) finalMsg;
            System.out.println("data from client: "
                    + byteBuf.toString(CharsetUtil.UTF_8));
        }
    });
    
    // 可以继续调用 ctx.channel().eventLoop().execute()
    // 将更多操作放入队列
    
    System.out.println("return right now.");
}
```

断点跟踪这个函数的执行，可以发现该耗时任务确实被放入的当前 NioEventLoop 的 taskQueue 中了。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/n3xax8gkhh.png)

对于第二种场景，举个例子，2.4 节的基于 Netty 编写的服务端的 Handler 中，假如 channelRead 方法中执行的过程并不需要立即执行，而是要定时执行，那么代码可以这样写：

```javascript
/**
 * 当通道有数据可读时执行
 *
 * @param ctx 上下文对象
 * @param msg 客户端发送的数据
 * @throws Exception
 */
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {

    final Object finalMsg = msg;

    // 通过 ctx.channel().eventLoop().schedule()将操作
    // 放入任务队列定时执行（5min 之后才进行处理）
    ctx.channel().eventLoop().schedule(new Runnable() {
        public void run() {

            ByteBuf byteBuf = (ByteBuf) finalMsg;
            System.out.println("data from client: "
                    + byteBuf.toString(CharsetUtil.UTF_8));
        }
    }, 5, TimeUnit.MINUTES);
    
    // 可以继续调用 ctx.channel().eventLoop().schedule()
    // 将更多操作放入队列

    System.out.println("return right now.");
}
```

断点跟踪这个函数的执行，可以发现该定时任务确实被放入的当前 NioEventLoop 的 scheduleTasjQueue 中了。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/umvnb65th9.png)

对于第三种场景，举个例子，比如在基于 Netty 构建的推送系统的业务线程中，要根据用户标识，找到对应的 SocketChannel 引用，然后调用 write 方法向该用户推送消息，这时候就会将这一 write 任务放在任务队列中，write 任务最终被异步消费。这种情形是对前两种情形的应用，且涉及的业务内容太多，不再给出示例代码，读者有兴趣可以自行完成，这里给出以下提示：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/deifw5aw9a.png)

## **6 Netty 的 Future 和 Promise**

Netty**对使用者提供的多数 IO 接口（即 Netty Channel 中的 IO 方法）**是异步的（**即都立即返回一个 Netty Future，而 IO 过程异步进行**），因此，调用者调用 IO 操作后是不能直接拿到调用结果的。要想得到 IO 操作结果，可以借助 Netty 的 Future（上面代码中的 ChannelFuture 就继承了 Netty Future，Netty Future 又继承了 JUC Future）查询执行状态、等待执行结果、获取执行结果等，使用过 JUC Future 接口的同学会非常熟悉这个机制，这里不再展开描述了。也可以通过 Netty Future 的 addListener()添加一个回调方法来异步处理 IO 结果，如下：

```javascript
// 启动客户端去连接服务器端
// 由于 bootstrap.connect()是一个异步操作，因此用.sync()等待
// 这个异步操作完成
final ChannelFuture channelFuture = bootstrap.connect(
        "127.0.0.1",
        8080).sync();

channelFuture.addListener(new ChannelFutureListener() {
    /**
     * 回调方法，上面的 bootstrap.connect()操作执行完之后触发
     */
    public void operationComplete(ChannelFuture future)
            throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("client has connected to server!");
            // TODO 其他处理
        } else {
            System.out.println("connect to serverfail!");
            // TODO 其他处理
        }
    }
});
```

Netty Future 提供的接口有：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/381exdr0hs.png)

> 注：会有一些资料给出这样的描述：“Netty 中所有的 IO 操作都是异步的”，这显然是错误的。Netty 基于 Java NIO，Java NIO 是同步非阻塞 IO。Netty 基于 Java NIO 做了封装，向使用者提供了异步特性的接口，因此本文说 Netty**对使用者提供的多数 IO 接口（即 Netty Channel 中的 IO 方法）**是异步的。例如在 io.netty.channel.ChannelOutboundInvoker（Netty Channel 的 IO 方法多继承于此）提供的多数 IO 接口都返回 Netty Future：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/i5klpt7tse.png)

Promise 是可写的 Future，Future 自身并没有写操作相关的接口，Netty 通过 Promise 对 Future 进行扩展，用于设置 IO 操作的结果。Future 继承了 Future，相关的接口定义如下图所示，相比于上图 Future 的接口，它多出了一些 setXXX 方法：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/2u0uaqagty.png)

Netty 发起 IO 写操作的时候，会创建一个新的 Promise 对象，例如调用 ChannelHandlerContext 的 write(Object object)方法时，会创建一个新的 ChannelPromise，相关代码如下：

```javascript
@Override
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}
......
@Override
public ChannelPromise newPromise() {
    return new DefaultChannelPromise(channel(), executor());
}
......
```



当 IO 操作发生异常或者完成时，通过 Promise.setSuccess()或者 Promise.setFailure()设置结果，并通知所有 Listener。关于 Netty 的 Future/Promise 的工作原理，我将在下一篇文章中进行源码级的解析。