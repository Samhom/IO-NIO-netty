## 传统 IO

### 一、java IO 系统

​	**IO 的最核心的概念就是流**，java 程序通过流来完成输入与输出。流是生产和消费信息的抽象，流通过 java 的输入输出系统与物理设备链接。尽管与它链接的物理设备不尽相同，所有流的系统具有相同的方式。这样，相同的输入输出类和方法适用于所有类型的外部设备，这意味着一个输入流能够抽象多种不同类型的输入：从磁盘文件、键盘、或者网络套接字。这样，一个输出流可以输出到控制台、磁盘文件或相连的网络。流是处理输入输出的一个洁净的方法，例如它不需要代码键盘和网络的不同。java 流的实现是在 java.io 包定义的类层次结构内部的。

​	输入输出时，数据是在通道中流动。所谓数据流指的是左右数据通信通道之中，数据的起点和终点。信息的通道就是一个数据流。只要是数据从一个地方“流”到另一个地方，这种数据流动的通道可以称为数据流。

​	java 中 的 IO 相关的类分为输入流和输出流，从结构上分为字符流（以字节为处理单位或称面向字节）和字节流。字节流的基础类分别是 InputStream 和 OutputStream 这两个抽象类，其输入输出操作由这两个类的子类来实现。字符流是后来新增的义字符为单位进行处理的流，其基础类是抽象类 Reader 和 Writer。

#### 1、阻塞模式的读/写数据的逻辑：

- open a steam
- while more information
- read/write information
- close the stream

#### 2、流的分类（重要）

##### 节点流-直接与设备打交道：

​	从特定的地方读写的流类，例如：磁盘或一块内存区域。

##### 过滤流：

​	使用节点流作为输入输出。过滤流使用一个已存在的输入流或输出流连接创建的。如可以对于直接设备打交道的 InputStream 进行一层包装，比如包装成一个带缓冲功能的 ByfferInputStream。接着还可以对已经包装过的流再次包装，这就是所谓的装饰器模式。

#### 3、InputStream 的类层次：

- FileInputStream（节点流）

- ByteArrayInputStream（节点流）

- **FilterInputStream**（过滤流）

  DataInputStream

  BufferedInputStream

  LineNumberInputStream

  PushbackInputStream

- ObjectInputStream（节点流）

- PipedInputStream（节点流）

- SequenceInputStream（节点流）

- StringBufferInputStream（节点流）

#### 4、InputStream 功能简介

​	InputStream 中包含一套字节输入流需要的方法，可以完成最基本的从输入流读入数据的功能。当 java 程序需要外设的数据时，可根据数据的不同形式，创建一个适当的 InputStream 子类类型的对象来完成与该外设的连接，然后再调用执行之歌流类对象的特定输入方法来实现对相应外设的输入操作。

​	InputStream 类子类对象自然也继承了 InputStream 的方法。常用的方法有：读数据的 read，获取输入流字节数的 available，定位输入位置指正的 skip、reset、Mark 等...

#### 5、OutputStream 简介

##### 几个基本的写方法：

- abstrat void write(int b)：往输入流中写入一个字节
- void write(byte[] b)：往输入流中写入数组 b 中的所有字节
- void write(byte[] b, int off, int len)：往输入流中写入数组 b 中从偏移量 off 开始的 len 个字节的数据
- void flush()：刷新输出流，强制缓冲区中的输出字节被写出
- void cloes()：关闭输出流，释放和这个流相关的系统资源

#### 6、OutputStream 的类层次：

- FIleOutputStream（节点流）

- ByteArrayOutputStream（节点流）

- **FilterOutputStream**（过滤流）

  DataOutputStream

  BufferedOutputStream

  PrintStream

- ObjectOutputStream（节点流）

- PipedOutputStream	（节点流）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200520010232834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

#### 7、Reader 的类层次

- BufferedReader （节点流）
  - LineNumberReader
- CharArrayReader（节点流）
- **FilterReder**（过滤流）
  - PushbackReader
- InputStreamReader（节点流）
  - FileReader
- PipedReader（节点流）
- StringReader（节点流）

#### 8、Writer 的类层次结构

- BufferedWriter（节点流）
- CharArrayWriter（节点流）
- **FilterWriter**（过滤流）
- OutputStreamWriter（节点流）
  - FileWriter
- PipedWriter（节点流）
- PrinteWriter（节点流）
- StringWriter（节点流）

### 二、java IO 的设计原则与使用的设计模式

​	java 的 IO 库提供了一个称作链接的机制，可以将一个流与另一个流首尾相接，形成一个流管道的链接。通过流的链接可以动态的增加流的功能，而这种功能的增加是通过组合一些流的基本功能而动态获取的。IO 库中 Decorator（装饰）模式的运用，给我提供了实现上的灵活性。

​	如 new C(new B(new A()))，就创建了同时具备 A、B、C 三个流功能的类。

​	装饰模式的特点是装饰对象和真实对象具有相同的接口，这样客户端对象就可以和真实对象相同的方式和装饰对象交互，且装饰对象包含一个真实对象的引用。

​	装饰对象接受来自客户端的请求，可以先/后增加一些附加功能，然后把请求转发给真实的对象。



