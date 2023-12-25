# Netty 概述

# 思维导图

![image.png](https://ucc.alicdn.com/pic/developer-ecology/b3fc6eb690464940b4a9b1100cfed5a2.png)

# 前言

本文主要讲述Netty框架的一些特性以及重要组件，希望看完之后能对Netty框架有一个比较直观的感受，希望能帮助读者快速入门Netty，减少一些弯路。

# 一、Netty概述

### **Netty 是什么**

1）Netty 是 JBoss 开源项目，是异步的、基于事件驱动的网络应用框架，它以高性能、高并发著称。所谓基于事件驱动，说得简单点就是 Netty 会根据客户端事件（连接、读、写等）做出响应。

2）Netty 主要用于开发基于 TCP 协议的网络 IO 程序（TCP/IP 是网络通信的基石，当然也是 Netty 的基石，Netty 并没有去改变这些底层的网络基础设施，而是在这之上提供更高层的网络基础设施），例如高性能[服务器](https://cloud.tencent.com/act/pro/promotion-cvm?from_column=20065&from=20065)段/客户端、P2P 程序等。

3）Netty 是基于 Java NIO 构建出来的，Java NIO 又是基于 Linux 提供的高性能 IO 接口/系统调用构建出来的。关于 Netty 在网络中的地位，下图可以很好地表达出来：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/lrq1k1tn2s.png)

### **Netty 的应用场景**

在互联网领域，Netty 作为异步高并发的网络组件，常常用于构建高性能 RPC 框架，以提升分布式服务群之间调用或者数据传输的并发度和速度。例如 Dubbo 的网络层就可以（但并非一定）使用 Netty。

一些[大数据基础](https://cloud.tencent.com/solution/bigdata?from_column=20065&from=20065)设施，比如 Hadoop，在处理海量数据的时候，数据在多个计算节点之中传输，为了提高传输性能，也采用 Netty 构建性能更高的网络 IO 层。

在游戏行业，Netty 被用于构建高性能的游戏交互服务器，Netty 提供了 TCP/UDP、HTTP 协议栈，方便开发者基于 Netty 进行私有协议的开发。

……

Netty 作为成熟的高性能异步通信框架，无论是应用在互联网分布式应用开发中，还是在[大数据](https://cloud.tencent.com/solution/bigdata?from_column=20065&from=20065)基础设施构建中，亦或是用于实现应用层基于公私协议的服务器等等，都有出色的表现，是一个极好的轮子。

### 非阻塞式 NIO

在这种模型中，服务器上一个线程处理多个连接，即多个客户端请求都会被注册到多路复用器（后文要讲的 Selector）上，多路复用器会轮训这些连接，轮训到连接上有 IO 活动就进行处理。NIO 降低了线程的需求量，提高了线程的利用率。Netty 就是基于 NIO 的（这里有一个问题：前文大力宣扬 Netty 是一个异步高性能网络应用框架，为何这里又说 Netty 是基于同步的 NIO 的？请读者跟着文章的描述找寻答案）。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/h9dtuaz1x5.png)

NIO 是面向缓冲区编程的，从缓冲区读取数据的时候游标在缓冲区中是可以前后移动的，这就增加了数据处理的灵活性。这和面向流的 BIO 只能顺序读取流中数据有很大的不同。

Java NIO 的非阻塞模式，使得一个线程从某个通道读取数据的时候，若当前有可用数据，则该线程进行处理，若当前无可用数据，则该线程不会保持阻塞等待状态，而是可以去处理其他工作（比如处理其他通道的读写）；同样，一个线程向某个通道写入数据的时候，一旦开始写入，该线程无需等待写完即可去处理其他工作（比如处理其他通道的读写）。这种特性使得一个线程能够处理多个客户端请求，而不是像 BIO 那样，一个线程只能处理一个请求。

#### NIO的缺点

既然 Java 提供了 NIO，为什么还要制造一个 Netty，文章[《NIO入门》](https://mp.weixin.qq.com/s/GfV9w2B0mbT7PmeBS45xLw?spm=a2c6h.12873639.article-detail.7.53064a61lP70dS)对 NIO 有比较详细的介绍，主要原因是 Java NIO 有以下几个缺点：

- NIO的类库和API繁杂，学习成本高，你需要熟练掌握Selector、ServerSocketChannel、SocketChannel、ByteBuffer等。
- 需要熟悉Java多线程编程。这是因为NIO编程涉及到Reactor模式，你必须对多线程和网络编程非常熟悉，才能写出高质量的NIO程序。
- 臭名昭著的epoll bug。它会导致Selector空轮询，最终导致CPU 100%。直到JDK1.7版本依然没得到根本性的解决。
- 支持多种主流协议。
- 预置多种编解码功能，支持用户开发私有协议。

从其中的几个关键词就能看出 Netty 的强大之处：零拷贝、可拓展事件模型；支持 TCP、UDP、HTTP、WebSocket 等协议；提供安全传输、压缩、大文件传输、编解码支持等等。

使用 NIO 构建 C/S 系统的 Java 编程组件是 Channel、Buffer、Selector。服务端示例代码为：

```java
public static void main(String[] args) throws IOException {
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    Selector selector = Selector.open();

    // 绑定端口
    serverSocketChannel.socket().bind(new InetSocketAddress(8080));

    // 设置 serverSocketChannel 为非阻塞模式
    serverSocketChannel.configureBlocking(false);

    // 注册 serverSocketChannel 到 selector，关注 OP_ACCEPT 事件
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

    while (true) {
        // 没有事件发生
        if (selector.select(1000) == 0) {
            continue;
        }

        // 有事件发生，找到发生事件的 Channel 对应的 SelectionKey 的集合
        Set<SelectionKey> selectionKeys = selector.selectedKeys();

        Iterator<SelectionKey> iterator = selectionKeys.iterator();
        while (iterator.hasNext()) {
            SelectionKey selectionKey = iterator.next();

            // 发生 OP_ACCEPT 事件，处理连接请求
            if (selectionKey.isAcceptable()) {
                SocketChannel socketChannel = serverSocketChannel.accept();

                // 将 socketChannel 也注册到 selector，关注 OP_READ
                // 事件，并给 socketChannel 关联 Buffer
                socketChannel.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));
            }

            // 发生 OP_READ 事件，读客户端数据
            if (selectionKey.isReadable()) {
                SocketChannel channel = (SocketChannel) selectionKey.channel();
                ByteBuffer buffer = (ByteBuffer) selectionKey.attachment();
                channel.read(buffer);

                System.out.println("msg form client: " + new String(buffer.array()));
            }

            // 手动从集合中移除当前的 selectionKey，防止重复处理事件
            iterator.remove();
        }
    }
}
```

#### **Java NIO API 简单回顾**

BIO 以流的方式处理数据，而 NIO 以缓冲区（也被叫做块）的方式处理数据，块 IO 效率比流 IO 效率高很多。BIO 基于字符流或者字节流进行操作，而 NIO 基于 Channel 和 Buffer 进行操作，数据总是从通道读取到缓冲区或者从缓冲区写入到通道。Selector 用于监听多个通道上的事件（比如收到连接请求、数据达到等等），因此使用单个线程就可以监听多个客户端通道。如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/c5ncwho7v8.png)

关于上图，再进行几点说明：

- 一个 Selector 对应一个处理线程
- 一个 Selector 上可以注册多个 Channel
- 每个 Channel 都会对应一个 Buffer（有时候一个 Channel 可以使用多个 Buffer，这时候程序要进行多个 Buffer 的分散和聚集操作），Buffer 的本质是一个内存块，底层是一个数组
- Selector 会根据不同的事件在各个 Channel 上切换
- Buffer 是双向的，既可以读也可以写，切换读写方向要调用 Buffer 的 flip()方法
- 同样，Channel 也是双向的，数据既可以流入也可以流出

#### **缓冲区（Buffer）**

缓冲区（Buffer）本质上是一个可读可写的内存块，可以理解成一个容器对象，Channel 读写文件或者网络都要经由 Buffer。在 Java NIO 中，Buffer 是一个顶层抽象类，它的常用子类有（前缀表示该 Buffer 可以存储哪种类型的数据）：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- DoubleBuffer
- FloatBuffer

涵盖了 Java 中除 boolean 之外的所有的基本数据类型。其中 ByteBuffer 支持类型化的数据存取，即可以往 ByteBuffer 中放 byte 类型数据、也可以放 char、int、long、double 等类型的数据，但读取的时候要做好类型匹配处理，否则会抛出 BufferUnderflowException。

另外，Buffer 体系中还有一个重要的 MappedByteBuffer（ByteBuffer 的子类），可以让文件内容直接在堆外内存中被修改，而如何同步到文件由 NIO 来完成。本文重点不在于此，有兴趣的可以去探究一下 MappedByteBuffer 的底层原理。

#### **通道（Channel）**

通道（Channel）是双向的，可读可写。在 Java NIO 中，Channel 是一个顶层接口，它的常用子类有：

- FileChannel：用于文件读写
- DatagramChannel：用于 UDP 数据包收发
- ServerSocketChannel：用于服务端 TCP 数据包收发
- SocketChannel：用于客户端 TCP 数据包收发

#### **选择器（Selector）**

选择器（Selector）是实现 IO 多路复用的关键，多个 Channel 注册到某个 Selector 上，当 Channel 上有事件发生时，Selector 就会取得事件然后调用线程去处理事件。也就是说只有当连接上真正有读写等事件发生时，线程才会去进行读写等操作，这就不必为每个连接都创建一个线程，一个线程可以应对多个连接。这就是 IO 多路复用的要义。

**Netty 的 IO 线程 NioEventLoop 聚合了 Selector**，可以同时并发处理成百上千的客户端连接，后文会展开描述。

在 Java NIO 中，Selector 是一个抽象类，它的常用方法有：

```java
public abstract class Selector implements Closeable {
    ......
    
    /**
     * 得到一个选择器对象
     */
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }
    ......

    /**
     * 返回所有发生事件的 Channel 对应的 SelectionKey 的集合，通过
     * SelectionKey 可以找到对应的 Channel
     */
    public abstract Set<SelectionKey> selectedKeys();
    ......
    
    /**
     * 返回所有 Channel 对应的 SelectionKey 的集合，通过 SelectionKey
     * 可以找到对应的 Channel
     */
    public abstract Set<SelectionKey> keys();
    ......
    
    /**
     * 监控所有注册的 Channel，当其中的 Channel 有 IO 操作可以进行时，
     * 将这些 Channel 对应的 SelectionKey 找到。参数用于设置超时时间
     */
    public abstract int select(long timeout) throws IOException;
    
    /**
    * 无超时时间的 select 过程，一直等待，直到发现有 Channel 可以进行
    * IO 操作
    */
    public abstract int select() throws IOException;
    
    /**
    * 立即返回的 select 过程
    */
    public abstract int selectNow() throws IOException;
    ......
    
    /**
    * 唤醒 Selector，对无超时时间的 select 过程起作用，终止其等待
    */
    public abstract Selector wakeup();
    ......
}
```

在上文的使用 Java NIO 编写的服务端示例代码中，服务端的工作流程为：

1）当客户端发起连接时，会通过 ServerSocketChannel 创建对应的 SocketChannel。

2）调用 SocketChannel 的注册方法将 SocketChannel 注册到 Selector 上，注册方法返回一个 SelectionKey，该 SelectionKey 会被放入 Selector 内部的 SelectionKey 集合中。该 SelectionKey 和 Selector 关联（即通过 SelectionKey 可以找到对应的 Selector），也和 SocketChannel 关联（即通过 SelectionKey 可以找到对应的 SocketChannel）。

4）Selector 会调用 select()/select(timeout)/selectNow() 方法对内部的 SelectionKey 集合关联的 SocketChannel 集合进行监听，找到有事件发生的 SocketChannel 对应的 SelectionKey。

5）通过 SelectionKey 找到有事件发生的 SocketChannel，完成数据处理。

以上过程的相关源码为：

```java
/**
* SocketChannel 继承 AbstractSelectableChannel
*/
public abstract class SocketChannel extends AbstractSelectableChannel implements ByteChannel,  ScatteringByteChannel,  GatheringByteChannel,  NetworkChannel {
    ......
}

public abstract class AbstractSelectableChannel extends SelectableChannel {
    ......
    /**
     * AbstractSelectableChannel 中包含注册方法，SocketChannel 实例
     * 借助该注册方法注册到 Selector 实例上去，该方法返回 SelectionKey
     */
    public final SelectionKey register(
        // 指明注册到哪个 Selector 实例
        Selector sel, 
        // ops 是事件代码，告诉 Selector 应该关注该通道的什么事件
        int ops,
        // 附加信息 attachment
        Object att) throws ClosedChannelException {
        ......
    }
    ......
}

public abstract class SelectionKey {
    ......

    /**
     * 获取该 SelectionKey 对应的 Channel
     */
    public abstract SelectableChannel channel();

    /**
     * 获取该 SelectionKey 对应的 Selector
     */
    public abstract Selector selector();
    ......
    
    /**
     * 事件代码，上面的 ops 参数取这里的值
     */
    public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;
    ......
    
    /**
     * 检查该 SelectionKey 对应的 Channel 是否可读
     */
    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

    /**
     * 检查该 SelectionKey 对应的 Channel 是否可写
     */
    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

    /**
     * 检查该 SelectionKey 对应的 Channel 是否已经建立起 socket 连接
     */
    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

    /**
     * 检查该 SelectionKey 对应的 Channel 是否准备好接受一个新的 socket 连接
     */
    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }

    /**
     * 添加附件（例如 Buffer）
     */
    public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    /**
     * 获取附件
     */
    public final Object attachment() {
        return attachment;
    }
    ......
}
```

下图用于辅助理解上面的过程和源码：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/4px2alvio0.png)

# 三、netty 架构图

![image.png](https://ucc.alicdn.com/pic/developer-ecology/15944ade0142471399997efd68e52948.png)

上面这张图就是在官网首页的架构图，我们从上到下分析一下。

> 绿色的部分**Core**核心模块，包括零拷贝、API库、可扩展的事件模型。
>
> 橙色部分**Protocol Support**协议支持，包括Http协议、webSocket、SSL(安全套接字协议)、谷歌Protobuf协议、zlib/gzip压缩与解压缩、Large File Transfer大文件传输等等。
>
> 红色的部分**Transport Services**传输服务，包括Socket、Datagram、Http Tunnel等等。

以上可看出Netty的功能、协议、传输方式都比较全，比较强大。

# 四、永远的 Hello Word

首先搭建一个HelloWord工程，先熟悉一下API，还有为后面的学习做铺垫。以下面这张图为依据：

![image.png](https://ucc.alicdn.com/pic/developer-ecology/cc27d56addd74e82b6b6b349c7f3769b.png)

## 4.1 引入Maven依赖

使用的版本是4.1.20，相对比较稳定的一个版本。

```
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.20.Final</version>
</dependency>
```

## 4.2 创建服务端启动类

```
public class MyServer {
    public static void main(String[] args) throws Exception {
        //创建两个线程组 boosGroup、workerGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            //创建服务端的启动对象，设置参数
            ServerBootstrap bootstrap = new ServerBootstrap();
            //设置两个线程组boosGroup和workerGroup
            bootstrap.group(bossGroup, workerGroup)
                //设置服务端通道实现类型    
                .channel(NioServerSocketChannel.class)
                //设置线程队列得到连接个数    
                .option(ChannelOption.SO_BACKLOG, 128)
                //设置保持活动连接状态    
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                //使用匿名内部类的形式初始化通道对象    
                .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            //给pipeline管道设置处理器
                            socketChannel.pipeline().addLast(new MyServerHandler());
                        }
                    });//给workerGroup的EventLoop对应的管道设置处理器
            System.out.println("java技术爱好者的服务端已经准备就绪...");
            //绑定端口号，启动服务端
            ChannelFuture channelFuture = bootstrap.bind(6666).sync();
            //对关闭通道进行监听
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

## 4.3 创建服务端处理器

```
/**
 * 自定义的Handler需要继承Netty规定好的HandlerAdapter
 * 才能被Netty框架所关联，有点类似SpringMVC的适配器模式
 **/
public class MyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //获取客户端发送过来的消息
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("收到客户端" + ctx.channel().remoteAddress() + "发送的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        //发送消息给客户端
        ctx.writeAndFlush(Unpooled.copiedBuffer("服务端已收到消息，并给你发送一个问号?", CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        //发生异常，关闭通道
        ctx.close();
    }
}
```

## 4.4 创建客户端启动类

```
public class MyClient {

    public static void main(String[] args) throws Exception {
        NioEventLoopGroup eventExecutors = new NioEventLoopGroup();
        try {
            //创建bootstrap对象，配置参数
            Bootstrap bootstrap = new Bootstrap();
            //设置线程组
            bootstrap.group(eventExecutors)
                //设置客户端的通道实现类型    
                .channel(NioSocketChannel.class)
                //使用匿名内部类初始化通道
                .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            //添加客户端通道的处理器
                            ch.pipeline().addLast(new MyClientHandler());
                        }
                    });
            System.out.println("客户端准备就绪，随时可以起飞~");
            //连接服务端
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6666).sync();
            //对通道关闭进行监听
            channelFuture.channel().closeFuture().sync();
        } finally {
            //关闭线程组
            eventExecutors.shutdownGracefully();
        }
    }
}
```

## 4.5 创建客户端处理器

```
public class MyClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        //发送消息到服务端
        ctx.writeAndFlush(Unpooled.copiedBuffer("歪比巴卜~茉莉~Are you good~马来西亚~", CharsetUtil.UTF_8));
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //接收服务端发送过来的消息
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("收到服务端" + ctx.channel().remoteAddress() + "的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
    }
}
```

## 4.6 测试

先启动服务端，再启动客户端，就可以看到结果：

MyServer打印结果:

![image.png](https://ucc.alicdn.com/pic/developer-ecology/92908e107d6a487bb930ab6cd6be269f.png)

MyClient打印结果：
![image.png](https://ucc.alicdn.com/pic/developer-ecology/419e8af300b24c9eaed71a76ddc2ddeb.png)

# 五、Netty的特性与重要组件

## 5.1 taskQueue任务队列

如果Handler处理器有一些长时间的业务处理，可以交给**taskQueue异步处理**。怎么用呢，请看代码演示：

```
public class MyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //获取到线程池eventLoop，添加线程，执行
        ctx.channel().eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                try {
                    //长时间操作，不至于长时间的业务操作导致Handler阻塞
                    Thread.sleep(1000);
                    System.out.println("长时间的业务处理");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

我们打一个debug调试，是可以看到添加进去的taskQueue有一个任务。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/23835a6ae2374897bf28a0b70fce9cc8.png)

## 5.2 scheduleTaskQueue延时任务队列

延时任务队列和上面介绍的任务队列非常相似，只是多了一个可延迟一定时间再执行的设置，请看代码演示：

```
ctx.channel().eventLoop().schedule(new Runnable() {
    @Override
    public void run() {
        try {
            //长时间操作，不至于长时间的业务操作导致Handler阻塞
            Thread.sleep(1000);
            System.out.println("长时间的业务处理");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
},5, TimeUnit.SECONDS);//5秒后执行
```

依然打开debug进行调试查看，我们可以有一个scheduleTaskQueue任务待执行中

![img](https://user-gold-cdn.xitu.io/2020/7/4/173194f21adfa111?w=705&h=214&f=png&s=17798)

## 5.3 Future异步机制

在搭建HelloWord工程的时候，我们看到有一行这样的代码：

```
ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6666);
```

很多操作都返回这个ChannelFuture对象，究竟这个ChannelFuture对象是用来做什么的呢？

ChannelFuture提供操作完成时一种异步通知的方式。一般在Socket编程中，等待响应结果都是同步阻塞的，而Netty则不会造成阻塞，因为ChannelFuture是采取类似观察者模式的形式进行获取结果。请看一段代码演示：

```
//添加监听器
channelFuture.addListener(new ChannelFutureListener() {
    //使用匿名内部类，ChannelFutureListener接口
    //重写operationComplete方法
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        //判断是否操作成功    
        if (future.isSuccess()) {
            System.out.println("连接成功");
        } else {
            System.out.println("连接失败");
        }
    }
});
```

## 5.4 Bootstrap与ServerBootStrap

Bootstrap和ServerBootStrap是Netty提供的一个创建客户端和服务端启动器的工厂类，使用这个工厂类非常便利地创建启动类，根据上面的一些例子，其实也看得出来能大大地减少了开发的难度。首先看一个类图：

![image.png](https://ucc.alicdn.com/pic/developer-ecology/40cf762660d9455eb6066119cf5eeb49.png)

可以看出都是继承于AbstractBootStrap抽象类，所以大致上的配置方法都相同。

一般来说，使用Bootstrap创建启动器的步骤可分为以下几步：

![image.png](https://ucc.alicdn.com/pic/developer-ecology/ae5c6ed3008d4323aaa817e9cb46437a.png)

### 5.4.1 group()

在上一篇文章[《Reactor模式》](https://mp.weixin.qq.com/s/vWbbn1qXRFVva8Y9yET18Q?spm=a2c6h.12873639.article-detail.8.53064a61lP70dS)中，我们就讲过服务端要使用两个线程组：

- bossGroup 用于监听客户端连接，专门负责与客户端创建连接，并把连接注册到workerGroup的Selector中。
- workerGroup用于处理每一个连接发生的读写事件。

一般创建线程组直接使用以下new就完事了：

```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

有点好奇的是，既然是线程组，那线程数默认是多少呢？深入源码：

```
    //使用一个常量保存
    private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        //NettyRuntime.availableProcessors() * 2，cpu核数的两倍赋值给常量
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }
    
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        //如果不传入，则使用常量的值，也就是cpu核数的两倍
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }
```

通过源码可以看到，默认的线程数是cpu核数的两倍。假设想自定义线程数，可以使用有参构造器：

```
//设置bossGroup线程数为1
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
//设置workerGroup线程数为16
EventLoopGroup workerGroup = new NioEventLoopGroup(16);
```

### 5.4.2 channel()

这个方法用于设置通道类型，当建立连接后，会根据这个设置创建对应的Channel实例。

![img](https://user-gold-cdn.xitu.io/2020/7/4/1731951ae3c43228?w=838&h=328&f=png&s=59491)

使用debug模式可以看到

![img](https://user-gold-cdn.xitu.io/2020/7/4/1731951f80b07785?w=495&h=212&f=png&s=16456)

通道类型有以下：

**NioSocketChannel**： 异步非阻塞的客户端 TCP Socket 连接。

**NioServerSocketChannel**： 异步非阻塞的服务器端 TCP Socket 连接。

> 常用的就是这两个通道类型，因为是异步非阻塞的。所以是首选。

OioSocketChannel： 同步阻塞的客户端 TCP Socket 连接。

OioServerSocketChannel： 同步阻塞的服务器端 TCP Socket 连接。

> 稍微在本地调试过，用起来和Nio有一些不同，是阻塞的，所以API调用也不一样。因为是阻塞的IO，几乎没什么人会选择使用Oio，所以也很难找到例子。我稍微琢磨了一下，经过几次报错之后，总算调通了。代码如下：

```
//server端代码，跟上面几乎一样，只需改三个地方
//这个地方使用的是OioEventLoopGroup
EventLoopGroup bossGroup = new OioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup)//只需要设置一个线程组boosGroup
        .channel(OioServerSocketChannel.class)//设置服务端通道实现类型

//client端代码，只需改两个地方
//使用的是OioEventLoopGroup
EventLoopGroup eventExecutors = new OioEventLoopGroup();
//通道类型设置为OioSocketChannel
bootstrap.group(eventExecutors)//设置线程组
        .channel(OioSocketChannel.class)//设置客户端的通道实现类型
```

NioSctpChannel： 异步的客户端 Sctp（Stream Control Transmission Protocol，流控制传输协议）连接。

NioSctpServerChannel： 异步的 Sctp 服务器端连接。

> 本地没启动成功，网上看了一些网友的评论，说是只能在linux环境下才可以启动。从报错信息看：SCTP not supported on this platform，不支持这个平台。因为我电脑是window系统，所以网友说的有点道理。

### 5.4.3 option()与childOption()

首先说一下这两个的区别。

option()设置的是服务端用于接收进来的连接，也就是boosGroup线程。

childOption()是提供给父管道接收到的连接，也就是workerGroup线程。

搞清楚了之后，我们看一下常用的一些设置有哪些：

SocketChannel参数，也就是childOption()常用的参数：

> **SO_RCVBUF** Socket参数，TCP数据接收缓冲区大小。
> **TCP_NODELAY** TCP参数，立即发送数据，默认值为Ture。
> **SO_KEEPALIVE** Socket参数，连接保活，默认值为False。启用该功能时，TCP会主动探测空闲连接的有效性。

ServerSocketChannel参数，也就是option()常用参数：

> **SO_BACKLOG** Socket参数，服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值，Windows为200，其他为128。

由于篇幅限制，其他就不列举了，大家可以去网上找资料看看，了解一下。

### 5.4.4 设置流水线(重点)

ChannelPipeline是Netty处理请求的责任链，ChannelHandler则是具体处理请求的处理器。实际上每一个channel都有一个处理器的流水线。

在Bootstrap中childHandler()方法需要初始化通道，实例化一个ChannelInitializer，这时候需要重写initChannel()初始化通道的方法，装配流水线就是在这个地方进行。代码演示如下：

```
//使用匿名内部类的形式初始化通道对象
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        //给pipeline管道设置自定义的处理器
        socketChannel.pipeline().addLast(new MyServerHandler());
    }
});
```

处理器Handler主要分为两种：

> ChannelInboundHandlerAdapter(入站处理器)、ChannelOutboundHandler(出站处理器)

入站指的是数据从底层java NIO Channel到Netty的Channel。

出站指的是通过Netty的Channel来操作底层的java NIO Channel。

**ChannelInboundHandlerAdapter处理器常用的事件有**：

1. 注册事件 fireChannelRegistered。
2. 连接建立事件 fireChannelActive。
3. 读事件和读完成事件 fireChannelRead、fireChannelReadComplete。
4. 异常通知事件 fireExceptionCaught。
5. 用户自定义事件 fireUserEventTriggered。
6. Channel 可写状态变化事件 fireChannelWritabilityChanged。
7. 连接关闭事件 fireChannelInactive。

**ChannelOutboundHandler处理器常用的事件有**：

1. 端口绑定 bind。
2. 连接服务端 connect。
3. 写事件 write。
4. 刷新时间 flush。
5. 读事件 read。
6. 主动断开连接 disconnect。
7. 关闭 channel 事件 close。

> 还有一个类似的handler()，主要用于装配parent通道，也就是bossGroup线程。一般情况下，都用不上这个方法。

### 5.4.5 bind()

提供用于服务端或者客户端绑定服务器地址和端口号，默认是异步启动。如果加上sync()方法则是同步。

有五个同名的重载方法，作用都是用于绑定地址端口号。不一一介绍了。

### 5.4.6 优雅地关闭EventLoopGroup

```
//释放掉所有的资源，包括创建的线程
bossGroup.shutdownGracefully();
workerGroup.shutdownGracefully();
```

会关闭所有的child Channel。关闭之后，释放掉底层的资源。

## 5.5 Channel

Channel是什么？不妨看一下官方文档的说明：

> A nexus to a network socket or a component which is capable of I/O operations such as read, write, connect, and bind

翻译大意：一种连接到网络套接字或能进行读、写、连接和绑定等I/O操作的组件。

如果上面这段说明比较抽象，下面还有一段说明：

> A channel provides a user:
>
> the current state of the channel (e.g. is it open? is it connected?),
> the configuration parameters of the channel (e.g. receive buffer size),
> the I/O operations that the channel supports (e.g. read, write, connect, and bind), and
> the ChannelPipeline which handles all I/O events and requests associated with the channel.

翻译大意：

channel为用户提供：

1. 通道当前的状态（例如它是打开？还是已连接？）
2. channel的配置参数（例如接收缓冲区的大小）
3. channel支持的IO操作（例如读、写、连接和绑定），以及处理与channel相关联的所有IO事件和请求的ChannelPipeline。

### 5.5.1 获取channel的状态

```
boolean isOpen(); //如果通道打开，则返回true
boolean isRegistered();//如果通道注册到EventLoop，则返回true
boolean isActive();//如果通道处于活动状态并且已连接，则返回true
boolean isWritable();//当且仅当I/O线程将立即执行请求的写入操作时，返回true。
```

以上就是获取channel的四种状态的方法。

### 5.5.2 获取channel的配置参数

获取单条配置信息，使用getOption()，代码演示：

```
ChannelConfig config = channel.config();//获取配置参数
//获取ChannelOption.SO_BACKLOG参数,
Integer soBackLogConfig = config.getOption(ChannelOption.SO_BACKLOG);
//因为我启动器配置的是128，所以我这里获取的soBackLogConfig=128
```

获取多条配置信息，使用getOptions()，代码演示：

```
ChannelConfig config = channel.config();
Map<ChannelOption<?>, Object> options = config.getOptions();
for (Map.Entry<ChannelOption<?>, Object> entry : options.entrySet()) {
    System.out.println(entry.getKey() + " : " + entry.getValue());
}
/**
SO_REUSEADDR : false
WRITE_BUFFER_LOW_WATER_MARK : 32768
WRITE_BUFFER_WATER_MARK : WriteBufferWaterMark(low: 32768, high: 65536)
SO_BACKLOG : 128
以下省略...
*/
```

### 5.5.3 channel支持的IO操作

**写操作**，这里演示从服务端写消息发送到客户端：

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.channel().writeAndFlush(Unpooled.copiedBuffer("这波啊，这波是肉蛋葱鸡~", CharsetUtil.UTF_8));
}
```

客户端控制台：

```
//收到服务端/127.0.0.1:6666的消息：这波啊，这波是肉蛋葱鸡~
```

**连接**操作，代码演示：

```
ChannelFuture connect = channelFuture.channel().connect(new InetSocketAddress("127.0.0.1", 6666));//一般使用启动器，这种方式不常用
```

**通过channel获取ChannelPipeline**，并做相关的处理：

```
//获取ChannelPipeline对象
ChannelPipeline pipeline = ctx.channel().pipeline();
//往pipeline中添加ChannelHandler处理器，装配流水线
pipeline.addLast(new MyServerHandler());
```

## 5.6 Selector

在NioEventLoop中，有一个成员变量selector，这是nio包的Selector，在之前[《NIO入门》](https://mp.weixin.qq.com/s/GfV9w2B0mbT7PmeBS45xLw)中，我已经讲过Selector了。

Netty中的Selector也和NIO的Selector是一样的，就是用于监听事件，管理注册到Selector中的channel，实现多路复用器。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/5fa70ed04e234fad9e524b3766051c4a.png)

## 5.7 PiPeline与ChannelPipeline

在前面介绍Channel时，我们知道可以在channel中装配ChannelHandler流水线处理器，那一个channel不可能只有一个channelHandler处理器，肯定是有很多的，既然是很多channelHandler在一个流水线工作，肯定是有顺序的。

于是pipeline就出现了，pipeline相当于处理器的容器。初始化channel时，把channelHandler按顺序装在pipeline中，就可以实现按序执行channelHandler了。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/e7bac501d86e4e75a897686d7bab40fe.png)

在一个Channel中，只有一个ChannelPipeline。该pipeline在Channel被创建的时候创建。ChannelPipeline包含了一个ChannelHander形成的列表，且所有ChannelHandler都会注册到ChannelPipeline中。

## 5.8 ChannelHandlerContext

在Netty中，Handler处理器是有我们定义的，上面讲过通过集成入站处理器或者出站处理器实现。这时如果我们想在Handler中获取pipeline对象，或者channel对象，怎么获取呢。

于是Netty设计了这个ChannelHandlerContext上下文对象，就可以拿到channel、pipeline等对象，就可以进行读写等操作。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/4c6e9319213b489bbfcc2d7697cf03b0.png)

通过类图，ChannelHandlerContext是一个接口，下面有三个实现类。

实际上ChannelHandlerContext在pipeline中是一个链表的形式。看一段源码就明白了：

```
//ChannelPipeline实现类DefaultChannelPipeline的构造器方法
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);
    //设置头结点head，尾结点tail
    tail = new TailContext(this);
    head = new HeadContext(this);
    
    head.next = tail;
    tail.prev = head;
}
```

下面我用一张图来表示，会更加清晰一点：
![image.png](https://ucc.alicdn.com/pic/developer-ecology/c77ea0ea4e554d65b61ee0a2eae78a0c.png)

## 5.9 EventLoopGroup

我们先看一下EventLoopGroup的类图：

![image.png](https://ucc.alicdn.com/pic/developer-ecology/7a95eeb933be4470acdc5f0f07afbc2a.png)

其中包括了常用的实现类NioEventLoopGroup。OioEventLoopGroup在前面的例子中也有使用过。

从Netty的架构图中，可以知道服务器是需要两个线程组进行配合工作的，而这个线程组的接口就是EventLoopGroup。

每个EventLoopGroup里包括一个或多个EventLoop，每个EventLoop中维护一个Selector实例。

### 5.9.1 轮询机制的实现原理

我们不妨看一段DefaultEventExecutorChooserFactory的源码：

```
private final AtomicInteger idx = new AtomicInteger();
private final EventExecutor[] executors;

@Override
public EventExecutor next() {
    //idx.getAndIncrement()相当于idx++，然后对任务长度取模
    return executors[idx.getAndIncrement() & executors.length - 1];
}
```

这段代码可以确定执行的方式是轮询机制，接下来debug调试一下：

![img](https://user-gold-cdn.xitu.io/2020/7/4/17319554d4546047?w=1241&h=520&f=png&s=82119)

它这里还有一个判断，如果线程数不是2的N次方，则采用取模算法实现。

```
@Override
public EventExecutor next() {
    return executors[Math.abs(idx.getAndIncrement() % executors.length)];
}
```

## 附1 **零拷贝技术**

以下讨论的是 Linux 系统下的 IO 过程。并且对于零拷贝技术的讲解采用了一种浅显易懂但能触及其本质的方式，因为这个话题，展开来讲实在是有太多的细节要关注。

在“**将本地磁盘中文件发送到网络中**”这一场景中，零拷贝技术是提升 IO 效率的一个利器，为了对比出零拷贝技术的优越性，下面依次给出使用**直接 IO 技术**、**内存映射文件技术**、**零拷贝技术**实现将本地磁盘文件发送到网络中的过程。

### 1）直接 IO 技术

使用直接 IO 技术实现文件传输的过程如下图所示。

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/5ddyb8dadm.png)

上图中，内核缓冲区是 Linux 系统的 Page Cahe。为了加快磁盘的 IO，Linux 系统会把磁盘上的数据以 Page 为单位缓存在操作系统的内存里，这里的 Page 是 Linux 系统定义的一个逻辑概念，一个 Page 一般为 4K。

可以看出，整个过程有四次数据拷贝，读进来两次，写回去又两次：磁盘-->内核缓冲区-->Socket 缓冲区-->网络。

直接 IO 过程使用的 Linux 系统 API 为：

```c++
ssize_t read(int filedes, void *buf, size_t nbytes);
ssize_t write(int filedes, void *buf, size_t nbytes);
```

等函数。

### 2）内存映射文件技术

使用内存映射文件技术实现文件传输的过程如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/p1gxc68biq.png)

可以看出，整个过程有三次数据拷贝，不再经过应用程序内存，直接在内核空间中从内核缓冲区拷贝到 Socket 缓冲区。

内存映射文件过程使用的 Linux 系统 API 为：

```c++
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

### 3）零拷贝技术

使用零拷贝技术，连内核缓冲区到 Socket 缓冲区的拷贝也省略了，如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/iglovjy6zr.png)

内核缓冲区到 Socket 缓冲区之间**并没有做数据的拷贝，只是一个地址的映射**。底层的网卡驱动程序要读取数据并发送到网络上的时候，看似读取的是 Socket 的缓冲区中的数据，**其实直接读的是内核缓冲区中的数据**。

零拷贝中所谓的“零”指的是内存中数据拷贝的次数为 0。

零拷贝过程使用的 Linux 系统 API 为：

```c++
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

在 JDK 中，提供的：

```java
FileChannel.transderTo(long position, long count, WritableByteChannel target);
```

方法实现了零拷贝过程，其中的第三个参数可以传入 SocketChannel 实例。例如客户端使用以上的零拷贝接口向服务器传输文件的代码为：

```java
public static void main(String[] args) throws IOException {
    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
    String fileName = "test.zip";

    // 得到一个文件 channel
    FileChannel fileChannel = new FileInputStream(fileName).getChannel();
    
    // 使用零拷贝 IO 技术发送
    long transferSize = fileChannel.transferTo(0, fileChannel.size(), socketChannel);
    System.out.println("file transfer done, size: " + transferSize);
    fileChannel.close();
}
```

## 附2 几种 Reactor 线程模式**

传统的 BIO 服务端编程采用“每线程每连接”的处理模型，弊端很明显，就是面对大量的客户端并发连接时，服务端的资源压力很大；并且线程的利用率很低，如果当前线程没有数据可读，它会阻塞在 read 操作上。这个模型的基本形态如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/z1601oozjw.jpeg)

BIO 服务端编程采用的是 Reactor 模式（也叫做 Dispatcher 模式，分派模式），Reactor 模式有两个要义：

1）基于 IO 多路复用技术，多个连接共用一个多路复用器，应用程序的线程无需阻塞等待所有连接，只需阻塞等待多路复用器即可。当某个连接上有新数据可以处理时，应用程序的线程从阻塞状态返回，开始处理这个连接上的业务。

2）基于线程池技术复用线程资源，不必为每个连接创建专用的线程，应用程序将连接上的业务处理任务分配给线程池中的线程进行处理，一个线程可以处理多个连接的业务。

下图反应了 Reactor 模式的基本形态：

![asdfff](https://ask.qcloudimg.com/http-save/yehe-6841114/2cfqtrrveh.jpeg)

Reactor 模式有两个核心组成部分：

1）Reactor（图中的 ServiceHandler）：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理线程来对 IO 事件做出反应。

2）Handlers（图中的 EventHandler）：处理线程执行处理方法来响应 I/O 事件，处理线程执行的是非阻塞操作。

Reactor 模式就是实现网络 IO 程序高并发特性的关键。它又可以分为单 Reactor 单线程模式、单 Reactor 多线程模式、主从 Reactor 多线程模式。

### **单 Reactor 单线程模式**

单 Reactor 单线程模式的基本形态如下：

**![img](https://ask.qcloudimg.com/http-save/yehe-6841114/3nuezqa9vz.jpeg)**

这种模式的基本工作流程为：

1）Reactor 通过 select 监听客户端请求事件，收到事件之后通过 dispatch 进行分发

2）如果事件是建立连接的请求事件，则由 Acceptor 通过 accept 处理连接请求，然后创建一个 Handler 对象处理连接建立后的后续业务处理。

3）如果事件不是建立连接的请求事件，则由 Reactor 对象分发给连接对应的 Handler 处理。

4）Handler 会完成 read-->业务处理-->send 的完整处理流程。

这种模式的优点是：模型简单，没有多线程、进程通信、竞争的问题，一个线程完成所有的事件响应和业务处理。当然缺点也很明显：

1）存在性能问题，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。

2）存在可靠性问题，若线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

单 Reactor 单线程模式使用场景为：客户端的数量有限，业务处理非常快速，比如 Redis 在业务处理的时间复杂度为 O(1)的情况。

### **单 Reactor 多线程模式**

单 Reactor 单线程模式的基本形态如下：

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/v98tr3a6ry.jpeg)

这种模式的基本工作流程为：

1）Reactor 对象通过 select 监听客户端请求事件，收到事件后通过 dispatch 进行分发。

2）如果事件是建立连接的请求事件，则由 Acceptor 通过 accept 处理连接请求，然后创建一个 Handler 对象处理连接建立后的后续业务处理。

3）如果事件不是建立连接的请求事件，则由 Reactor 对象分发给连接对应的 Handler 处理。Handler 只负责响应事件，不做具体的业务处理，Handler 通过 read 读取到请求数据后，会分发给后面的 Worker 线程池来处理业务请求。

4）Worker 线程池会分配独立线程来完成真正的业务处理，并将处理结果返回给 Handler。Handler 通过 send 向客户端发送响应数据。

这种模式的优点是可以充分的利用多核 cpu 的处理能力，缺点是多线程数据共享和控制比较复杂，Reactor 处理所有的事件的监听和响应，在单线程中运行，面对高并发场景还是容易出现性能瓶颈。

### **主从 Reactor 多线程模式**

主从 Reactor 多线程模式的基本形态如下（第二章图片是 JUC 作者 Doug Lea 在《Scalable IO in Java》中给出的示意图，两张图表达的含义一样）：

![asdfasdf](https://ask.qcloudimg.com/http-save/yehe-6841114/7mf0rtmuyh.jpeg)

![img](https://ask.qcloudimg.com/http-save/yehe-6841114/ruoy35pfpo.jpeg)

针对单 Reactor 多线程模型中，Reactor 在单个线程中运行，面对高并发的场景易成为性能瓶颈的缺陷，主从 Reactor 多线程模式让 Reactor 在多个线程中运行（分成 MainReactor 线程与 SubReactor 线程）。这种模式的基本工作流程为：

1）Reactor 主线程 MainReactor 对象通过 select 监听客户端连接事件，收到事件后，通过 Acceptor 处理客户端连接事件。

2）当 Acceptor 处理完客户端连接事件之后（与客户端建立好 Socket 连接），MainReactor 将连接分配给 SubReactor。（即：MainReactor 只负责监听客户端连接请求，和客户端建立连接之后将连接交由 SubReactor 监听后面的 IO 事件。）

3）SubReactor 将连接加入到自己的连接队列进行监听，并创建 Handler 对各种事件进行处理。

4）当连接上有新事件发生的时候，SubReactor 就会调用对应的 Handler 处理。

5）Handler 通过 read 从连接上读取请求数据，将请求数据分发给 Worker 线程池进行业务处理。

6）Worker 线程池会分配独立线程来完成真正的业务处理，并将处理结果返回给 Handler。Handler 通过 send 向客户端发送响应数据。

7）一个 MainReactor 可以对应多个 SubReactor，即一个 MainReactor 线程可以对应多个 SubReactor 线程。

这种模式的优点是：

1）MainReactor 线程与 SubReactor 线程的数据交互简单职责明确，MainReactor 线程只需要接收新连接，SubReactor 线程完成后续的业务处理。

2）MainReactor 线程与 SubReactor 线程的数据交互简单， MainReactor 线程只需要把新连接传给 SubReactor 线程，SubReactor 线程无需返回数据。

3）多个 SubReactor 线程能够应对更高的并发请求。

这种模式的缺点是编程复杂度较高。但是由于其优点明显，在许多项目中被广泛使用，包括 Nginx、Memcached、Netty 等。

这种模式也被叫做服务器的 **1+M+N** 线程模式，即使用该模式开发的服务器包含一个（或多个，1 只是表示相对较少）连接建立线程+M 个 IO 线程+N 个业务处理线程。这是业界成熟的服务器程序设计模式。