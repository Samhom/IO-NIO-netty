## NIO—Selector

### 一、基于 IO 的网络编程的基本流程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525222236184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

​	这种方式如果处理较少的客户端链接时具有很大的优势，但是一旦客户端链接很多的时候就会生成很多的处理线程，过多的线程除了占用内存之外，还会导致 CPU 上下文的频繁切换，性能会大打折扣。

### 二、NIO 的网络编程模型

​	NIO 是服务端能够使服务端能够使用一个线程来处理所有客户端的请求连接。这也就是引申除了**异步**的概念。

​	Selector 是 NIO 重要的概念。NIO 也即是结合 Event 来实现这种异步编程模型。

​	下面来构建一种模式，使一个线程来监听 5 个客户端的链接，看下使用 NIO 的编程模式：

```java 
public staitc void main(String[] args) {
    int[] ports = new int[5];
    ports[0] = 5001;
}
```

### 一、Selector 概念剖析

​	**Selector** 一般称 为**选择器** ，它是 Java NIO 核心组件中的一个，用于检查一个或多个 NIO Channel（通道）的状态是否处于可读、可写。如此可以实现单线程管理多个 channels，也就是可以管理多个网络链接。

![Selector(选择器)](https://user-gold-cdn.xitu.io/2018/5/15/16363f5338f36c54?w=636&h=260&f=png&s=23373)

#### 1. Selector 的创建

```java
Selector selector = Selector.open();

public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}

public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                    if (loadProviderFromProperty())
                        return provider;
                    if (loadProviderAsService())
                        return provider;
                    provider = sun.nio.ch.DefaultSelectorProvider.create();
                    return provider;
                }
            });
    }
}

// 底层都是位于 sun.io 包下边的未开源的代码实现

```

#### 2. 注册 Channel 到 Selector

```java
// Channel 必须是非阻塞的。
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, Selectionkey.OP_READ);
```

​	FileChannel 不适用 Selector，因为 FileChannel 不能切换为非阻塞模式，更准确的来说是因为 FileChannel 没有继承 SelectableChannel。Socket channel 可以正常使用。

​	**SelectableChannel 抽象类** 有一个 **configureBlocking（）** 方法用于使通道处于阻塞模式或非阻塞模式。

```java
abstract SelectableChannel configureBlocking(boolean block)
```

​	SelectableChannel 抽象类的 configureBlocking（）  方法是由  AbstractSelectableChannel抽象类实现的，**SocketChannel、ServerSocketChannel、DatagramChannel** 都是直接继承了 **AbstractSelectableChannel** 抽象类 。

register() 方法的第二个参数。这是一个“ **interest集合** ”，意思是在通过 Selector 监听 Channel 时 对什么事件感兴趣。可以监听四种不同类型的事件：

- **Connect**

  SelectionKey.OP_CONNECT

- **Accept**

  SelectionKey.OP_ACCEPT

- **Read**

  SelectionKey.OP_READ

- **Write**

  SelectionKey.OP_WRITE

​	通道触发了一个事件意思是该事件已经就绪。比如某个Channel成功连接到另一个服务器称为“ **连接就绪** ”。一个Server Socket Channel准备好接收新进入的连接称为“ **接收就绪** ”。一个有数据可读的通道可以说是“ **读就绪** ”。等待写数据的通道可以说是“ **写就绪** ”。

如果你对不止一种事件感兴趣，使用或运算符即可，如下：

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

#### 3. SelectionKey 介绍

​	一个 SelectionKey 键表示了一个特定的通道对象和一个特定的选择器对象之间的注册关系。

```java
key.attachment(); //返回 SelectionKey 的 attachment，attachment 可以在注册 channel 的时候指定。
key.channel(); // 返回该 SelectionKey 对应的 channel。
key.selector(); // 返回该 SelectionKey 对应的 Selector。
key.interestOps(); // 返回代表需要 Selector 监控的 IO 操作的 bit mask
key.readyOps(); // 返回一个 bit mask，代表在相应 channel 上可以进行的 IO 操作。
```

##### key.interestOps()：

​	可以通过以下方法来判断Selector是否对Channel的某种事件感兴趣

```java
int interestSet = selectionKey.interestOps(); 
boolean isInterestedInAccept = (interestSet & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT；
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite = interestSet & SelectionKey.OP_WRITE;
```

##### key.readyOps()：

​	ready 集合是通道已经准备就绪的操作的集合。JAVA 中定义以下几个方法用来检查这些操作是否就绪：

```java
// 创建 ready 集合的方法
int readySet = selectionKey.readyOps();
// 检查这些操作是否就绪的方法
key.isAcceptable();// 是否可读，是返回 true
boolean isWritable()：// 是否可写，是返回 true
boolean isConnectable()：// 是否可连接，是返回 true
boolean isAcceptable()：// 是否可接收，是返回 true
```

##### 从 SelectionKey 访问 Channel 和 Selector 如下：

```java
Channel channel = key.channel();
Selector selector = key.selector();
key.attachment();
```

​	可以将一个对象或者更多信息附着到 SelectionKey 上，这样就能方便的识别某个给定的通道。例如，可以附加与通道一起使用的 Buffer，或是包含聚集数据的某个对象。使用方法如下：

```java
key.attach(theObject);
Object attachedObj = key.attachment();
```

还可以在用 register() 方法向 Selector 注册 Channel 的时候附加对象。如：

```java
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

### 4. 从 Selector 中选择 channel

​	选择器维护注册过的通道的集合，并且这种注册关系都被封装在 SelectionKey 当中。

Selector 维护的三种类型 SelectionKey 集合：

- **已注册的键的集合(Registered key set)**

  ​	所有与选择器关联的通道所生成的键的集合称为已经注册的键的集合。并不是所有注册过的键都仍然有效。这个集合通过 **keys()** 方法返回，并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引发java.lang.UnsupportedOperationException。

- **已选择的键的集合(Selected key set)**

  ​	表示的是注册在当前的 Selector 上的所有感兴趣事件发生的  Channel 注册对应的键的集合。这个集合通过 **selectedKeys()** 方法返回。这个集合是已注册的键的集合的子集。

- **已取消的键的集合(Cancelled key set)**

  ​	已注册的键的集合的子集，这个集合包含了 **cancel()** 方法被调用过的键(这个键已经被无效化)，但它们还没有被注销。这个集合是选择器对象的私有成员，因而无法直接访问。

  ​	当键被取消（ 可以通过**isValid( )** 方法来判断）时，它将被放在相关的选择器的已取消的键的集合里。注册不会立即被取消，但键会立即失效。当再次调用 **select( )** 方法时（或者一个正在进行的 select() 调用结束时），已取消的键的集合中的被取消的键将被清理掉，并且相应的注销也将完成。通道会被注销，而新的 SelectionKey 将被返回。当通道关闭时，所有相关的键会自动取消（记住，一个通道可以被注册到多个选择器上）。当选择器关闭时，所有被注册到该选择器的通道都将被注销，并且相关的键将立即被无效化（取消）。一旦键被无效化，调用它的与选择相关的方法就将抛出 CancelledKeyException。

##### select()方法介绍：

​	在刚初始化的 Selector 对象中，这三个集合都是空的。 **通过 Selector的 select（）方法可以选择已经准备就绪的通道** （这些通道包含你感兴趣的的事件）。比如对读就绪的通道感兴趣，那么select（）方法就会返回读事件已经就绪的那些通道。下面是 Selector 几个重载的 select() 方法：

- int select()：阻塞到至少有一个通道在你注册的事件上就绪了。
- int select(long timeout)：和 select()一样，但最长阻塞时间为 timeout 毫秒。
- int selectNow()：非阻塞，只要有通道就绪就立刻返回。

​	select() 方法返回的 int 值表示有多少通道已经就绪，是自上次调用 select() 方法后有多少通道变成就绪状态。之前在 select() 调用时进入就绪的通道不会在本次调用中被记入，而在前一次 selec() 调用进入就绪但现在已经不在处于就绪的通道也不会被记入。

​	例如：首次调用 select() 方法，如果有一个通道变成就绪状态，返回了 1，若再次调用select() 方法，如果另一个通道就绪了，它会再次返回 1。如果对第一个就绪的 channel 没有做任何操作，现在就有两个就绪的通道，但在每次 select() 方法调用之间，只有一个通道就绪了。

​	一旦调用 select() 方法，并且返回值不为 0 时，则可以通过调用 Selector 的 selectedKeys() 方法来访问已选择键集合 。如下：

```java
Set selectedKeys = selector.selectedKeys();
```

​	进而可以放到和某 SelectionKey 关联的 Selector 和 Channel。如下所示：

```java
int select = selector.select();
if (select == 0) {
    return;
}
Set selectedKeys = selector.selectedKeys();
Iterator keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.
    } else if (key.isConnectable()) {
        // a connection was established with a remote server.
    } else if (key.isReadable()) {
        // a channel is ready for reading
    } else if (key.isWritable()) {
        // a channel is ready for writing
    }
    keyIterator.remove();
}
```

#### 5. 停止选择的方法

​	选择器执行选择的过程，系统底层会依次询问每个通道是否已经就绪，这个过程可能会造成调用线程进入阻塞状态，那么我们有以下三种方式可以唤醒在 select() 方法中阻塞的线程。

- **wakeup() 方法** ：

  ​	通过调用 Selector 对象的 wakeup() 方法让处在阻塞状态的 select() 方法立刻返回。该方法使得选择器上的第一个还没有返回的选择操作立即返回。如果当前没有进行中的选择操作，那么下一次对 select() 方法的一次调用将立即返回。

- **close()方法** ：

  ​	通过 close() 方法关闭 Selector，该方法使得任何一个在选择操作中阻塞的线程都被唤醒，类似 wakeup()，同时使得注册到该 Selector 的所有 Channel 被注销，所有的键将被取消，但是 Channel 本身并不会关闭。

### 四、模板代码

**一个服务端的模板代码：**

有了模板代码，编写程序时，大多数时间都是在模板代码中添加相应的业务代码。

```java
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.socket().bind(new InetSocketAddress("localhost", 8080));
ssc.configureBlocking(false);

Selector selector = Selector.open();
ssc.register(selector, SelectionKey.OP_ACCEPT);

while(true) {
    int readyNum = selector.select();
    if (readyNum == 0) {
        continue;
    }

    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> it = selectedKeys.iterator();

    while(it.hasNext()) {
        SelectionKey key = it.next();
        if (key.isAcceptable()) {
            // 接受连接
        } else if (key.isReadable()) {
            // 通道可读
        } else if (key.isWritable()) {
            // 通道可写
        }
        it.remove();
    }
}
```

### 五、客户端与服务端简单交互实例

#### 服务端：

```java
package selector;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class WebServer {
    public static void main(String[] args) {
        try {
            ServerSocketChannel ssc = ServerSocketChannel.open();
            ssc.socket().bind(new InetSocketAddress("127.0.0.1", 8000));
            ssc.configureBlocking(false);

            Selector selector = Selector.open();
            // 注册 channel，并且指定感兴趣的事件是 Accept
            ssc.register(selector, SelectionKey.OP_ACCEPT);

            ByteBuffer readBuff = ByteBuffer.allocate(1024);
            ByteBuffer writeBuff = ByteBuffer.allocate(128);
            writeBuff.put("received".getBytes());
            writeBuff.flip();

            while (true) {
                int nReady = selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> it = keys.iterator();

                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    it.remove();

                    if (key.isAcceptable()) {
                        // 创建新的连接，并且把连接注册到 selector 上，而且，
                        // 声明这个 channel 只对读操作感兴趣。
                        SocketChannel socketChannel = ssc.accept();
                        socketChannel.configureBlocking(false);
                        socketChannel.register(selector, SelectionKey.OP_READ);
                    } else if (key.isReadable()) {
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        readBuff.clear();
                        socketChannel.read(readBuff);

                        readBuff.flip();
                        System.out.println("received : " + new String(readBuff.array()));
                        key.interestOps(SelectionKey.OP_WRITE);
                    } else if (key.isWritable()) {
                        writeBuff.rewind();
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        socketChannel.write(writeBuff);
                        key.interestOps(SelectionKey.OP_READ);
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### 客户端：

```java
package selector;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class WebClient {
    public static void main(String[] args) throws IOException {
        try {
            SocketChannel socketChannel = SocketChannel.open();
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 8000));

            ByteBuffer writeBuffer = ByteBuffer.allocate(32);
            ByteBuffer readBuffer = ByteBuffer.allocate(32);

            writeBuffer.put("hello".getBytes());
            writeBuffer.flip();

            while (true) {
                writeBuffer.rewind();
                socketChannel.write(writeBuffer);
                readBuffer.clear();
                socketChannel.read(readBuffer);
            }
        } catch (IOException e) {
        }
    }
}
```

**运行结果：**

先运行服务端，再运行客户端，服务端会不断收到客户端发送过来的消息：

![运行结果](https://user-gold-cdn.xitu.io/2018/5/16/1636720b53ff3a72?w=1090&h=217&f=png&s=15376)