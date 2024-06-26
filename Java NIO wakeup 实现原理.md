# Java NIO wakeup 实现原理

# 导语

最近在阅读`netty`源码时，很好奇Java NIO中`Selector`的`wakeup()`方法是如何唤醒`selector`的，于是决定深扒一下`wakeup`机制的实现原理，相信对学习`NIO`是大有裨益的。

# wakeup语义

众所周知，`selector.select()`是阻塞的，通常情况下，只有注册在selector上的channel有事件就绪时，select()才会从阻塞中被唤醒，处理就绪事件。那么，当selector上的channel无就绪事件时，如果想要唤醒阻塞在select()操作上的线程去处理一些别的工作，该如何实现呢？事实上Selector提供了这样的API：

```java
public abstract Selector wakeup();
解释
```

wakeup()实现的功能：

- 如果一个线程在调用select()或select(long)方法时被阻塞，调用`wakeup()`会使线程立即从阻塞中唤醒；如果调用`wakeup()`期间没有select操作，下次调用select相关操作会立即返回，不会执行`poll()`，包括调用selectNow()。
- 在Select期间，多次调用wakeup()与调用一次效果是一样的。

注意：如果调用wakeup()期间没有select操作，后续若先调用一次selectNow()，再次调用select()则会导致阻塞。

# wakeup实现机制

以上描述了wakeup()的功能，那么JAVA NIO中是如何实现这个机制的呢？下面以windows环境为例，结合源码来探究这个问题。

通常我们会使用`Selector.open()`方法创建一个选择器对象，`SelectorProvider`负责根据不同操作系统来返回不同的实现类，windows平台就返回`WindowsSelectorProvider`，然后再调用其`openSelector()`。

```java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
解释
```

从WindowsSelectorProvider的openSelector()可知，其作用是创建一个WindowsSelectorImpl对象：

```java
public AbstractSelector openSelector() throws IOException {
    return new WindowsSelectorImpl(this);
}
解释
```

WindowsSelectorImpl就是Selector接口的最终实现类，我们来看看其构造方法都做了什么：

```java
WindowsSelectorImpl(SelectorProvider sp) throws IOException {
    super(sp);
    pollWrapper = new PollArrayWrapper(INIT_CAP);
    wakeupPipe = Pipe.open();
    wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

    SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
    (sink.sc).socket().setTcpNoDelay(true);//禁用Nagle算法
    wakeupSinkFd = ((SelChImpl)sink).getFDVal();

    pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
}
解释
```

从源码可知，实例化WindowsSelectorImpl时，会调用Pipe.open()创建一个管道实例wakeupPipe，并从wakeupPipe中获取wakeupSourceFd和wakeupSinkFd两个文件描述符，wakeupSourceFd为read端FD，wakeupSinkFd为write端FD，然后将wakeupSourceFd加入pollWrapper中。

我们知道，pollWrapper的作用是保存当前selector对象上注册的FD，当调用Selector的select()方法时，会将pollWrapper的内存地址传递给内核，由内核负责轮训pollWrapper中的FD，一旦有事件就绪，将事件就绪的FD传递回用户空间，阻塞在select()的线程就会被唤醒。将wakeupSourceFd加入pollWrapper中，表示selector也需要关注wakeupSourceFd上发生的事件，而谁会处理该事件呢？我们先了解下Pipe吧。

从广义上说，管道就是一个用来在两个实体之间单向传输数据的导管。在Unix系统中，管道被用来连接一个进程的输出和另一个进程的输入。Java使用Pipe类实现了一个管道范例，只不过它创建的管道是进程内（JVM进程内部)而非进程间使用的。

Pipe实现的管道由一个可写的SinkChannel和一个可读的SourceChannel组成，这两个Channel的远端是连接起来的，使得一旦将一些字节写入到SinkChannel，就可以在SourceChannel按写入顺序读取这些字节。下面我们看看SinkChannel和SourceChannel类继承结构图：

![img](https://static.oschina.net/uploads/space/2017/0812/165335_lzzO_2663573.png)

从图中我们知道几点：

- SinkChannel和SourceChannel都扩展了AbstractSelectableChannel，因此都支持被注册到一个Selector上；
- SourceChannel只实现了ReadableByteChannel，因此只支持读操作；同时实现了ScatteringByteChannel，具有将通道中的数据分散到多个缓冲区的能力（矢量I/O）；
- SinkChannel只实现了WritableByteChannel，因此只支持写操作；同时实现了GatheringByteChannel，具有将多个缓冲区的数据聚集到该通道的能力（矢量I/O）；
- SinkChannel和SourceChannel的实现类都实现了SelChImpl，因此都能获取通道相关联的文件描述符FD；
- SourceChannel和SinkChannel内部通过聚合SocketChannel来完成读和写相关的操作。

下面我们继续分析Pipe.open()的实现；

Pipe.open()最终会创建一个PipeImpl实例：

```java
PipeImpl(final SelectorProvider sp) throws IOException {
    try {
        AccessController.doPrivileged(new Initializer(sp));
    } catch (PrivilegedActionException x) {
        throw (IOException)x.getCause();
    }
}
解释
```

PipeImpl构造方法中会创建一个内部类Initializer实例，并调用它的run方法：

```java
public Void run() throws IOException {
     LoopbackConnector connector = new LoopbackConnector();
     connector.run();
    ......
}
解释
```

Initializer的run方法则会创建内部LoopbackConnector的实例，并调用它的run方法，其主要作用是建立一条本地环回连接，看实现：

```java
public void run() {
    ServerSocketChannel ssc = null;
    SocketChannel sc1 = null;
    SocketChannel sc2 = null;

    try {
        // 环回地址
        InetAddress lb = InetAddress.getByName("127.0.0.1");
        assert(lb.isLoopbackAddress());
        InetSocketAddress sa = null;
        for(;;) {
            // 绑定ServerSocketChannel到环回地址上的一个端口
            if (ssc == null || !ssc.isOpen()) {
                ssc = ServerSocketChannel.open();
                ssc.socket().bind(new InetSocketAddress(lb, 0));
                sa = new InetSocketAddress(lb, ssc.socket().getLocalPort());
            }

            //建立连接
            sc1 = SocketChannel.open(sa);
            ByteBuffer bb = ByteBuffer.allocate(8);
            long secret = rnd.nextLong();
            bb.putLong(secret).flip();
            sc1.write(bb);

            // 获取连接并校验合法性
            sc2 = ssc.accept();
            bb.clear();
            sc2.read(bb);
            bb.rewind();
            if (bb.getLong() == secret)
                break;
            sc2.close();
            sc1.close();
        }

        // 创建source通道和sink通道
        source = new SourceChannelImpl(sp, sc1);
        sink = new SinkChannelImpl(sp, sc2);
    } catch (IOException e) {
        try {
            if (sc1 != null)
                sc1.close();
            if (sc2 != null)
                sc2.close();
        } catch (IOException e2) {}
        ioe = e;
    } finally {
        try {
            if (ssc != null)
                ssc.close();
        } catch (IOException e2) {}
    }
}
解释
```

run方法完成的功能：

- 使用本地环回地址“127.0.0.1”创建一个InetAddress实例lb。“127.0.0.1”是一个保留地址，主要用于环回测试，也就是说，目的地址为环回地址的IP数据包永远都不会出现在任何网络中；
- 创建一个ServerSocketChannelImpl实例ssc，为该通道绑定一个唯一的文件描述符FD；
- 使用lb和0号端口创建一个InetSocketAddress实例，并将该实例绑定到服务端socket通道上。这里使用了系统预留的0号端口，主要是为了避免写死端口号，操作系统会从动态端口号范围内搜索接下来可以使用的端口号作为服务端socket通道的监听端口；
- 使用lb和ssc上绑定的端口号创建一个InetSocketAddress实例sa，再用sa创建一个SocketChannel实例，并为该通道绑定一个唯一的文件描述符FD；
- 客户端socket通道创建成功后会调用connect尝试建立socket连接，由于当前处于阻塞模式，因此connect会阻塞直到成功建立连接或发生IO错误；
- 创建一个8字节的ByteBuffer对象，填充一个随机long值，然后将缓冲区的数据写入通道sc1；
- 调用ssc的accept()方法，accept方法会创建一个新的SocketChannel实例，并绑定一个唯一的文件描述符FD，然后使用这个SocketChannel实例读取数据；
- 比较发送的数据和接收的数据是否相等，若相等，使用sc1创建SourceChannelImpl实例作为管道的source端，使用sc2创建SinkChannelImpl实例作为管道的sink端；
- 最后调用close()关闭ServerSocketChannel，这样ServerSocketChannel就不会接受新的连接，同时释放绑定在该通道上的FD。

到此，一个管道被成功建立，这个管道的两端为两个通道，SourceChannel作为read端，而SinkChannel为write端，两个通道之间通过TCP进行连接，这样使得在SinkChannel端写入的数据SourceChannel端可以立马读取。

Java中Pipe实现的管道仅用于在同一个Java虚拟机内部传输数据。实际应用中，使用管道在线程间传输数据也是一种不错的方案，它为我们提供了良好的封装性。

现在，回到WindowsSelectorImpl的构造方法中，我们知道，创建一个Selector实例时，还会创建一个管道Pipe实例，并将管道source端wakeupSourceFd加入pollWrapper中，作为第一个注册到Selector的FD，并设置感兴趣的事件为Net.POLLIN，表示对可读事件感兴趣。当Selector在轮训pollWrapper中的FD时，如果wakeupSourceFd发生read事件，那么Selector就会被唤醒，这就是wakeup()的实现原理。看wakeup()实现：

```java
public Selector wakeup() {
    synchronized (interruptLock) {
        if (!interruptTriggered) {
            setWakeupSocket();
            interruptTriggered = true;
        }
    }
    return this;
}
解释
```

首先判断interruptTriggered，如果为True，立即返回；如果为False，调用setWakeupSocket(),并将interruptTriggered设置为true。下面看setWakeupSocket()的实现：

```java
private void setWakeupSocket() {
    this.setWakeupSocket0(this.wakeupSinkFd);
}
private native void setWakeupSocket0(int wakeupSinkFd);
解释
```

传入管道sink端的wakeupSinkFd，然后调用底层的setWakeupSocket0方法，下面从openjdk8源文件WindowsSelectorImpl.c找到setWakeupSocket0的实现：

```java
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd)
{
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}
解释
```

该函数的主要作用是向pipe的sink端写入了一个字节，这样pipe的source端文件描述符立即就会处于就绪状态，select()方法将立即从阻塞中返回，这样就完成了唤醒selector的功能。

wakeup()中使用interruptTriggered来判断是否执行唤醒操作。因此，在select期间，多次调用wakeup()产生的效果与调用一次是一样的，因为后面的调用将不会满足唤醒条件。如果调用wakeup()期间没有select操作，当调用wakeup()之后，interruptTriggered被设置为true，pipe的source端wakeupSourceFd 就会处于就绪状态。如果此时调用select相关操作时，会调用resetWakeupSocket 方法，resetWakeupSocket 首先会调用本地方法resetWakeupSocket0读取wakeup()中发送的数据，再将interruptTriggered设置为false，最后doSelect将会立即返回0，而不会调用poll操作。

为什么说将一些字节写入到SinkChannel后，SourceChannel就可以立即按写入顺序读取这些字节？

这是因为我们在WindowsSelectorImpl构造方法中将TCP参数TCP_NODELAY设置为了true。该参数的主要作用是禁用Nagle算法，当sink端写入1字节数据时，将立即发送，而不必等到将较小的包组合成较大的包再发送，这样source端就可以立马读取数据。

下面附上windows环境下Selector的实现原理图帮助理解阻塞与唤醒的原理（图片来源网络）：

![img](https://static.oschina.net/uploads/space/2017/0812/234333_CS1h_2663573.jpg)

# 总结

本文主要介绍了windows环境下wakeup()的实现原理，它通过一个可写的SinkChannel和一个可读的SourceChannel组成的pipe来实现唤醒的功能，而Linux环境则使用其本身的Pipe来实现唤醒功能。无论windows还是linux，wakeup的思想是完全一致的，只不过windows没有Pipe这种信号通知的机制，所以通过TCP来实现了Pipe，建立了一对自己和自己的loopback的TCP连接来发送信号。请注意，每创建一个Selector对象，都会创建一个Pipe实例，这会导致消耗两个文件描述符FD和两个端口号，实际开发中需注意端口号和文件描述符的限制。