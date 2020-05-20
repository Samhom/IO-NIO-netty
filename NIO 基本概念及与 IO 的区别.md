## NIO 基本概念及与 IO 的区别

先看一个简单例子：

```java
public static void main(String[] args) {
    IntBuffer buffer = IntBuffer.allocate(10);
    for (int i = 0; i < 10; i++) {
        int randomNumber = new SourceRandom.nextInt();
        buffer.put(randomNumber);
    }
    // 转换操作方式之前一定要进行 flip 操作
    buffer.flip();
    while (buffer.hasRemaining()) {
        System.out.println(buffer.get());
    }
}
```

### 一、NIO 的组件间关系图示

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200520224317396.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

### 二、IO 与 NIO 的区别

1. IO 中最核心的一个概念的是流，也就是 IO 是面向流的编程。对一个流来说要么是输入流要么是输出流，不能同时是输入与输出流。
2. NIO 有三个核心概念：Selector、Channel、Buffer。在 NIO 中是面向块（Buffer）编程的。在下边的图示中，一个 Selector 在其关联的 Channel 上来回切换，根据什么切换呢，就是事件，其实 NIO 中事件是也是一个比较重要的概念，什么时候链接建立了、数据发送成功了、服务器返回了数据，这些都是一个个的事件，在编写程序的时候就是通过事件来判断相应的响应。比如 ChannelA 要写入到其关联的 Buffer 当中，这时候就发生了读事件，Selector 就会选择 ChannelA 进行相应的处理。处理的过程当中还可以切换到其他的 Channel 上去。
3. 可以将 NIO 中的 Channel 理解为 IO 中的 Stream。NIO 中的 Buffer 就是一块儿内存，在使用额时候需要分配内存大小，数据的读与写都是通过 Buffer 来实现的。且在 NIO 中一个缓冲区可以作为一个读写的区域，这个功能就是通过 flip 方法实现读写切换的。
4. IO 中从文件中读入到 Stream 中，然后从 Stream 中就可以读入到程序当中，但是 NIO 中与文件关联的是 Channel，也就是说文件是先读到 Channel 中的，然后在把 Channel 中的数据写入到 Buffer 当中，之后才能从 Buffer 中读取数据到程序当中，在这中间程序是不能直接从 Channel 中读取数据的。
5. NIO 包含了 java 中的除了布尔值的 7 大种类基本数据类型的 Buffer 类型。由于 Channel 是双向的，因此它能够反映出底层操作系统的真实情况：在 Linux 中，底层操作系统的通道是双向的。

### 三、再看两个例子

```java
public static void main(String[] args) {
    FileInputStream fi = new FileInputStream("NioTest1.txt");
    FileChannel fc = fi.getChannel();
    ByteBuffer buffer = ByteBuffer.allocate(512);
    fc.read(buffer); // 将 channel 中的数据写入到 buffer 中
    
    // 转换操作方式之前一定要进行 flip 操作
    buffer.flip();
    
    while (buffer.remaining() > 0) {
        byte b = buffer.get();
        System.out.println((char) b);
    }
    
    fi.close();
}
```

```java
public static void main(String[] args) {
    FileOutStream fo = new FileOutStream("NioTest1.txt");
    FileChannel fc = fo.getChannel();
    ByteBuffer buffer = ByteBuffer.allocate(512);
    
    byte[] message = "hellor world, lilei".getBytes();
    for (int i = 0; i < message.length; i++) {
        buffer.put(message[i]);
    }
    
    // 转换操作方式之前一定要进行 flip 操作
    buffer.flip();
    fc.write(buffer)
    
    fo.close();
}
```

















