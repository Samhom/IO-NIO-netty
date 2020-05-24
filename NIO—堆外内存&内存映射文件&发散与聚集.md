## NIO—Channel 解读

### 1、从文件中读取内容然后写入到另一个文件中：

```java
	public static void main(String[] args) throws IOException {
        FileInputStream fileInputStream = new FileInputStream("fileIn.txt");
        FileOutputStream fileOutputStream = new FileOutputStream("fileOut.txt");

        FileChannel channelIn = fileInputStream.getChannel();
        FileChannel channelOut = fileOutputStream.getChannel();

        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        while (true) {
            byteBuffer.clear();// 该行代码被注释掉会发生什么
            int read = channelIn.read(byteBuffer);
            if (-1 == read) {
                break;
            }
            byteBuffer.flip();
            channelOut.write(byteBuffer);
        }
        fileInputStream.close();
        fileOutputStream.close();
    }
```

### 2、ByteBuffer 多类型数据写入与读取：

```java
	public static void main(String[] args) throws IOException {
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        byteBuffer.putInt(12);
        byteBuffer.putLong(222L);
        byteBuffer.putShort((short) 9);
        byteBuffer.putChar('a');
        byteBuffer.putDouble(9.88);
        byteBuffer.putFloat(833);

        byteBuffer.flip();

        // 读取的顺序要和写入的顺序一致，可以通过此方式进行协议的指定
        System.out.println(byteBuffer.getInt());
        System.out.println(byteBuffer.getLong());
        System.out.println(byteBuffer.getShort());
        System.out.println(byteBuffer.getChar());
        System.out.println(byteBuffer.getDouble());
        System.out.println(byteBuffer.getFloat());
    }
```

### 3、ByteBuffer 的分片截取 slice：

```java
public static void main(String[] args) throws IOException {
    ByteBuffer byteBuffer = ByteBuffer.allocate(10);

    for (int i = 0; i < byteBuffer.capacity(); i++) {
        byteBuffer.put((byte) i);
    }

    // 在调用 clice 方法前需要指定截取的起始位置
    byteBuffer.position(2);
    byteBuffer.limit(6);

    ByteBuffer slice = byteBuffer.slice();

    for (int i = 0; i < slice.capacity(); i++) {
        byte a = slice.get(i);
        a *= 2;
        slice.put(i, a);
    }

    byteBuffer.position(0);
    byteBuffer.limit(byteBuffer.capacity());

    while (byteBuffer.hasRemaining()) {
        System.out.println(byteBuffer.get());
    }
}
```

ByteBuffer 中抽象方法 slice 的说明如下：

```java
/**
 * Creates a new byte buffer whose content is a shared subsequence of
 * this buffer's content.
   新建一个共享内容序列的 buffer。
 *
 * <p> The content of the new buffer will start at this buffer's current
 * position.  Changes to this buffer's content will be visible in the new
 * buffer, and vice versa; the two buffers' position, limit, and mark
 * values will be independent.
   新的 buffer 的从当前 buffer 当前的 position 位置开始，两个 buffer 的内容的修改对
   彼此是可见的，两者的 position、limit、mark 是独立的。
 *
 * <p> The new buffer's position will be zero, its capacity and its limit
 * will be the number of bytes remaining in this buffer, and its mark
 * will be undefined.  The new buffer will be direct if, and only if, this
 * buffer is direct, and it will be read-only if, and only if, this buffer
 * is read-only.  </p>
   
 *
 * @return  The new byte buffer
 */
public abstract ByteBuffer slice();
```

slice 方法的实现在 HeapByteBuffer 中：

```
public ByteBuffer slice() {
    return new HeapByteBuffer(hb,
                                    -1,
                                    0,
                                    this.remaining(),
                                    this.remaining(),
                                    this.position() + offset);
}
```

​	从以上可知，hb 也就是存放元素的引用是共用的，然后新建了一个 HeapByteBuffer，还可以知道，mark,pos,lim,cap 属性是独立的。

### 4、readOnlyBuffer 只读 buffer

​	只读 buffer 的使用场景一般是传输给调用方时，并不希望对方对当期 buffer 内容修改时使用。ByteBuffer 的只读 buffer 是 **HeapByteBufferR**。

```java
public static void main(String[] args) throws IOException {
    ByteBuffer byteBuffer = ByteBuffer.allocate(10);
    for (int i = 0; i < byteBuffer.capacity(); i++) {
        byteBuffer.put((byte) i);
    }
    ByteBuffer readOnlyBuffer = byteBuffer.asReadOnlyBuffer();

    System.out.println(readOnlyBuffer.getClass());
}
```

ByteBuffer 中 asReadOnlyBuffer 抽象方法的定义：

```java
/**
 * Creates a new, read-only byte buffer that shares this buffer's
 * content.
   创建一个新的只读 buffer，与当前 buffer 内容（底层数组）共享。
 *
 * <p> The content of the new buffer will be that of this buffer.  Changes
 * to this buffer's content will be visible in the new buffer; the new
 * buffer itself, however, will be read-only and will not allow the shared
 * content to be modified.  The two buffers' position, limit, and mark
 * values will be independent.
   大致意思就是只读 buffer 不允许修改内容。
 *
 * <p> The new buffer's capacity, limit, position, and mark values will be
 * identical to those of this buffer.
   新 buffer 的 capacity, limit, position, and mark 与当前一致。
 *
 * <p> If this buffer is itself read-only then this method behaves in
 * exactly the same way as the {@link #duplicate duplicate} method.  </p>
 *
 * @return  The new, read-only byte buffer
 */
public abstract ByteBuffer asReadOnlyBuffer();
```

asReadOnlyBuffer 方法的实现在 HeapByteBuffer 中：

```java
public ByteBuffer asReadOnlyBuffer() {
	return duplicate();
}

public ByteBuffer duplicate() {
    // 可知这里新建了一个 HeapByteBufferR 对象
	return new HeapByteBufferR(hb,
		this.markValue(),
        this.position(),
        this.limit(),
        this.capacity(),
        offset);
}

protected HeapByteBufferR(byte[] buf, int mark, int pos, int lim, int cap, int off) {
	super(buf, mark, pos, lim, cap, off);
    // isReadOnly 属性是定义在 ByteBuffer 中的
    this.isReadOnly = true;
}
```

### 5、ByteBuffer.wrap 

​	ByteBuffer 除了有 allocate 方法来创建一个 buffer 外，还有一个 wrap 方法，该方法允许用户传入一个数组，如此以来，修改 buffer 的内容的方法就又了两种途径。

```java
public static ByteBuffer wrap(byte[] array) {
    return wrap(array, 0, array.length);
}

public static ByteBuffer wrap(byte[] array,
                                    int offset, int length)
    {
        try {
            return new HeapByteBuffer(array, offset, length);
        } catch (IllegalArgumentException x) {
            throw new IndexOutOfBoundsException();
        }
    }
```

### 6、非堆内存分配 buffer

#### 一、使用堆外内存的原因：

1. 对垃圾回收停顿的改善

   ​	因为 full gc 意味着彻底回收，彻底回收时，垃圾收集器会对所有分配的堆内内存进行完整的扫描，这意味着一个重要的事实——这样一次垃圾收集对 Java 应用造成的影响，跟堆的大小是成正比的。过大的堆会影响 Java 应用的性能。如果使用堆外内存的话，堆外内存是直接受操作系统管理( 而不是虚拟机 )。这样做的结果就是能保持一个较小的堆内内存，以减少垃圾收集对应用的影响。

2. 零拷贝的实现

   ​	在某些场景下可以提升程序 I/O 操纵的性能。少去了将数据从堆内内存拷贝到堆外内存的步骤，以此来实现零拷贝的功能。

DirectByteBuffer 是通过虚引用(Phantom Reference)来实现堆外内存的释放的。

#### 二、linux 的内核态和用户态

![img](https://upload-images.jianshu.io/upload_images/4235178-7c5ca2cb236fd2eb.png?imageMogr2/auto-orient/strip|imageView2/2/w/752/format/webp)

![img](https://upload-images.jianshu.io/upload_images/4235178-2393d0797135217b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

​	java 中通过 JNI 调用的 native 方法实际上就是从用户态切换到了内核态的一种方式。并且通过该系统调用使用操作系统所提供的功能。

​	为什么需要用户进程(位于用户态中)要通过系统调用(Java 中即使 JNI)来调用内核态中的资源？

​	intel cpu 提供 Ring0-Ring3 四种级别的运行模式，Ring0 级别最高，Ring3 最低。Linux 使用了 Ring3 级别运行用户态，Ring0 作为内核态。Ring3 状态不能访问 Ring0 的地址空间，包括代码和数据。因此用户态是没有权限去操作内核态的资源的，它只能通过系统调用外完成用户态到内核态的切换，然后在完成相关操作后再由内核态切换回用户态。

![img](https://upload-images.jianshu.io/upload_images/4235178-fc2ae3eac18813d3.png?imageMogr2/auto-orient/strip|imageView2/2/w/726/format/webp)

#### 三、在 linux 中内核态的权限是最高的，那么在内核态的场景下，操作系统是可以访问任何一个内存区域的，所以操作系统是可以访问到 Java 堆的这个内存区域的。那为什么操作系统不直接访问 Java 堆内的内存区域？

​	这是因为 JNI 方法访问的内存区域是一个已经确定了的内存区域地质，那么该内存地址指向的是 Java 堆内内存的话，那么如果在操作系统正在访问这个内存地址的时候，Java 在这个时候进行了 GC 操作，而 GC 操作会涉及到数据的移动操作，GC 经常会进行先标志在压缩的操作。即将可回收的空间做标志，然后清空标志位置的内存，然后会进行一个压缩，压缩就会涉及到对象的移动，移动的目的是为了腾出一块更加完整、连续的内存空间，以容纳更大的新对象，数据的移动会使 JNI 调用的数据错乱。所以 JNI 调用的内存是不能进行 GC 操作的。

#### 四、JNI 调用的内存是不能进行 GC 操作的，那该如何解决？

​	1、堆内内存与堆外内存之间数据拷贝的方式(并且在将堆内内存拷贝到堆外内存的过程 JVM 会保证不会进行 GC 操作)：比如我们要完成一个从文件中读数据到堆内内存的操作，即 FileChannelImpl.read(HeapByteBuffer)。这里实际上 File I/O 会将数据读到**堆外内存**中，然后堆外内存再将数据拷贝到**堆内内存**，这样我们就读到了文件中的内存。而写操作则反之，我们会将堆内内存的数据线写到对堆外内存中，然后操作系统会将堆外内存的数据写入到文件中。

![img](https://upload-images.jianshu.io/upload_images/4235178-f94db8df14023550.png?imageMogr2/auto-orient/strip|imageView2/2/w/1194/format/webp)

```java
    static int read(FileDescriptor var0, ByteBuffer var1, long var2, NativeDispatcher var4) throws IOException {
        if (var1.isReadOnly()) {
            throw new IllegalArgumentException("Read-only buffer");
        } else if (var1 instanceof DirectBuffer) {
            return readIntoNativeBuffer(var0, var1, var2, var4);
        } else {
            // 分配临时的堆外内存
            ByteBuffer var5 = Util.getTemporaryDirectBuffer(var1.remaining());

            int var7;
            try {
                // File I/O 操作会将数据读入到堆外内存中
                int var6 = readIntoNativeBuffer(var0, var5, var2, var4);
                var5.flip();
                if (var6 > 0) {
                    // 将堆外内存的数据拷贝到堆外内存中
                    var1.put(var5);
                }

                var7 = var6;
            } finally {
                // 里面会调用 DirectBuffer.cleaner().clean() 来释放临时的堆外内存
                Util.offerFirstTemporaryDirectBuffer(var5);
            }

            return var7;
        }
    }
```

​	2、直接使用堆外内存，如 DirectByteBuffer：这种方式是直接在堆外分配一个内存(即，native memory)来存储数据，程序通过 JNI 直接将数据读/写到堆外内存中。因为数据直接写入到了堆外内存中，所以这种方式就不会再在 JVM 管控的堆内再分配内存来存储数据了，也就不存在堆内内存和堆外内存数据拷贝的操作了。这样在进行 I/O 操作时，只需要将这个堆外内存地址传给 JNI 的 I/O 的函数就好了。

#### 五、DirectByteBuffer 源码解析：

​	ByteBuffer.allocateDirect 操作会创建一个 DirectByteBuffer 的类：

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

DirectByteBuffer 构造函数如下：

```java
// 堆外内存分配
DirectByteBuffer(int cap) {                   // package-private
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    // 保留总分配内存(按页分配)的大小和实际内存的大小
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 通过 unsafe 的 native 方法分配堆外内存，并返回堆外内存的基地址
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        // address 属性放在 Buffer 抽象类中，这里就是讲 address 这个引用指向一块内存边界上去
        // 之所以将 address 属性升级放在 Buffer 中，是为了在 JNI 调用 GetDirectBufferAddress 时提升它调用的速率。
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 构建 Cleaner 对象用于跟踪 DirectByteBuffer 对象的垃圾回收，以实现当 DirectByteBuffer 被垃圾回收时，堆外内存也会被释放
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

##### a、Bits.reserveMemory(size, cap) 方法

```java
    static void reserveMemory(long size, int cap) {

        if (!memoryLimitSet && VM.isBooted()) {
            maxMemory = VM.maxDirectMemory();
            memoryLimitSet = true;
        }

        // optimist!
        if (tryReserveMemory(size, cap)) {
            return;
        }

        final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();

        // retry while helping enqueue pending Reference objects
        // which includes executing pending Cleaner(s) which includes
        // Cleaner(s) that free direct buffer memory
        while (jlra.tryHandlePendingReference()) {
            if (tryReserveMemory(size, cap)) {
                return;
            }
        }

        // trigger VM's Reference processing
        System.gc();

        // a retry loop with exponential back-off delays
        // (this gives VM some time to do it's job)
        boolean interrupted = false;
        try {
            long sleepTime = 1;
            int sleeps = 0;
            while (true) {
                if (tryReserveMemory(size, cap)) {
                    return;
                }
                if (sleeps >= MAX_SLEEPS) {
                    break;
                }
                if (!jlra.tryHandlePendingReference()) {
                    try {
                        Thread.sleep(sleepTime);
                        sleepTime <<= 1;
                        sleeps++;
                    } catch (InterruptedException e) {
                        interrupted = true;
                    }
                }
            }

            // no luck
            throw new OutOfMemoryError("Direct buffer memory");

        } finally {
            if (interrupted) {
                // don't swallow interrupts
                Thread.currentThread().interrupt();
            }
        }
    }
```

其中，如果系统中内存( 即，堆外内存 )不够的话：

```java
        final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();

        // retry while helping enqueue pending Reference objects
        // which includes executing pending Cleaner(s) which includes
        // Cleaner(s) that free direct buffer memory
        while (jlra.tryHandlePendingReference()) {
            if (tryReserveMemory(size, cap)) {
                return;
            }
        }
```

​	jlra.tryHandlePendingReference() 会触发一次非堵塞的 Reference#tryHandlePending(false)。该方法会将已经被 JVM 垃圾回收的 DirectBuffer 对象的堆外内存释放。

 因为在 Reference 的静态代码块中定义了：

```java
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean tryHandlePendingReference() {
                return tryHandlePending(false);
            }
        });
```

如果在进行一次堆外内存资源回收后，还不够进行本次堆外内存分配的话，则

```java
        // trigger VM's Reference processing
        System.gc();
```

​	System.gc() 会触发一个 full gc，当然前提是你没有显示的设置 -XX:+DisableExplicitGC 来禁用显式 GC。并且你需要知道，调用 System.gc() 并不能够保证 full gc 马上就能被执行。

​	 所以在后面打代码中，会进行最多 9 次尝试，看是否有足够的可用堆外内存来分配堆外内存。并且每次尝试之前，都对延迟等待时间，已给 JVM 足够的时间去完成 full gc 操作。如果 9 次尝试后依旧没有足够的可用堆外内存来分配本次堆外内存，则抛出 OutOfMemoryError("Direct buffer memory”) 异常：

![img](https://upload-images.jianshu.io/upload_images/4235178-6da0d60191992f59.png?imageMogr2/auto-orient/strip|imageView2/2/w/449/format/webp)

​	这里之所以用使用 full gc 的很重要的一个原因是：System.gc() 会对新生代的老生代都会进行内存回收，这样会比较彻底地回收 DirectByteBuffer 对象以及他们关联的堆外内存.DirectByteBuffer对象本身其实是很小的，但是它后面可能关联了一个非常大的堆外内存，因此我们通常称之为**冰山对象**。

​	如果有大量的 DirectByteBuffer 对象移到了 old，但是又一直没有做 cms gc 或者 full gc，而只进行 ygc，那么我们的物理内存可能被慢慢耗光，但是我们还不知道发生了什么，因为 heap 明明剩余的内存还很多(前提是我们禁用了 System.gc – JVM 参 数 DisableExplicitGC)。

​	总的来说，Bits.reserveMemory(size, cap) 方法在可用堆外内存不足以分配给当前要创建的堆外内存大小时，会实现以下的步骤来尝试完成本次堆外内存的创建：

 ① 触发一次非堵塞的 Reference#tryHandlePending(false)。该方法会将已经被 JVM 垃圾回收的 DirectBuffer 对象的堆外内存释放。

 ② 如果进行一次堆外内存资源回收后，还不够进行本次堆外内存分配的话，则进行 System.gc()。System.gc() 会触发一个 full gc，但你需要知道，调用 System.gc() 并不能够保证 full gc 马上就能被执行。所以在后面打代码中，会进行最多 9 次尝试，看是否有足够的可用堆外内存来分配堆外内存。并且每次尝试之前，都对延迟等待时间，已给 JVM 足够的时间去完成 full gc 操作。

 注意，如果你设置了 -XX:+DisableExplicitGC，将会禁用显示 GC，这会使 System.gc() 调用无效。
 ③ 如果 9 次尝试后依旧没有足够的可用堆外内存来分配本次堆外内存，则抛出 OutOfMemoryError("Direct buffer memory”) 异常。

##### b、堆外内存回收

​	Cleaner 是 PhantomReference 的子类，并通过自身的 next 和 prev 字段维护的一个双向链表。PhantomReference 的作用在于跟踪垃圾回收过程，并不会对对象的垃圾回收过程造成任何的影响。

​	所以 cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); 用于对当前构造的 DirectByteBuffer 对象的垃圾回收过程进行跟踪。

​	当 DirectByteBuffer 对象从 pending 状态 ——> enqueue 状态时，会触发 Cleaner 的 clean()，而 Cleaner的 clean() 的方法会实现通过 unsafe 对堆外内存的释放。

![img](https://upload-images.jianshu.io/upload_images/4235178-792afac32aefd061.png?imageMogr2/auto-orient/strip|imageView2/2/w/713/format/webp)

![img](https://upload-images.jianshu.io/upload_images/4235178-07eaab88f1d02927.png?imageMogr2/auto-orient/strip|imageView2/2/w/750/format/webp)

 ​	虽然 Cleaner 不会调用到 Reference.clear()，但 Cleaner 的 clean() 方法调用了 remove(this)，即将当前 Cleaner 从 Cleaner 链表中移除，这样当 clean() 执行完后，Cleaner就是一个无引用指向的对象了，也就是可被 GC 回收的对象。

##### c、thunk 方法：

![img](https://upload-images.jianshu.io/upload_images/4235178-ebeffa00197df134.png?imageMogr2/auto-orient/strip|imageView2/2/w/515/format/webp)

##### d、通过配置参数的方式来回收堆外内存

​	同时我们可以通过 -XX:MaxDirectMemorySize 来指定最大的堆外内存大小，当使用达到了阈值的时候将调用 System.gc() 来做一次 full gc，以此来回收掉没有被使用的堆外内存。

#### 六、MappedByteBuffer 源码解读

​	DirectByteBuffer 继承自 MappedByteBuffer。

​	java io 操作中通常采用 BufferedReader，BufferedInputStream 等带缓冲的 IO 类处理大文件，不过 java nio 中引入了一种基于 MappedByteBuffer 操作大文件的方式，其读写性能极高。

​	从继承结构上看，MappedByteBuffer 继承自 ByteBuffer，内部维护了一个逻辑地址 address。也叫做内存映射缓冲区。通过磁盘上文件在内存中的映射，当修改内存中的数据时，会被映射到磁盘上。也就是允许通过内存直接访问文件。可以将整个文件或者文件的一部分映射到内存当中，接着是由操作系统完成内存页到文件中的写入操作。用于内存映射文件的内存时堆外内存。

##### 1、先看一个既简单例子：

```java
public static void main(String[] args) throws IOException {
    RandomAccessFile randomAccessFile = new RandomAccessFile("aaa.txt", "rw");
    MappedByteBuffer buffer = randomAccessFile.getChannel().map(FileChannel.MapMode.READ_ONLY, 0, 10);
    buffer.put(0, (byte) 'a');
    buffer.put(1, (byte) 'b');
    randomAccessFile.close();
}
```

channel.map：

```java
// 表示映射后可读写的方式，position 表示开始映射的下标，size 表示映射的大小
public abstract MappedByteBuffer map(MapMode mode,
                                     long position, long size)
    throws IOException;

public static class MapMode {

    public static final MapMode READ_ONLY
        = new MapMode("READ_ONLY");

    public static final MapMode READ_WRITE
        = new MapMode("READ_WRITE");

    public static final MapMode PRIVATE
        = new MapMode("PRIVATE");
    ...
```

##### 2、FileLock

FileLock 可以实现对文件操作的锁定。

```
public static void main(String[] args) throws IOException {
    RandomAccessFile randomAccessFile = new RandomAccessFile("aaa.txt", "rw");
    FileChannel channel = randomAccessFile.getChannel();

    FileLock lock = channel.lock();
    System.out.println(lock.isValid());
    System.out.println(lock.isShared());
    lock.release();
    
    channel.close();
    randomAccessFile.close();
}
```

##### 3、接着看下该类的说明：

```java
/**
 * A direct byte buffer whose content is a memory-mapped region of a file.
   文件内存映射的缓冲区
 *
 * <p> Mapped byte buffers are created via the {@link
 * java.nio.channels.FileChannel#map FileChannel.map} method.  This class
 * extends the {@link ByteBuffer} class with operations that are specific to
 * memory-mapped file regions.
   通过 FileChannel.map 方法创建映射缓冲区
 *
 * <p> A mapped byte buffer and the file mapping that it represents remain
 * valid until the buffer itself is garbage-collected.
   映射缓冲区和其代表的文件映射内存是在当前 buffer 对象被垃圾回收掉以后进行释放的。
 *
 * <p> The content of a mapped byte buffer can change at any time, for example
 * if the content of the corresponding region of the mapped file is changed by
 * this program or another.  Whether or not such changes occur, and when they
 * occur, is operating-system dependent and therefore unspecified.
 *
 * <a name="inaccess"></a><p> All or part of a mapped byte buffer may become
 * inaccessible at any time, for example if the mapped file is truncated.  An
 * attempt to access an inaccessible region of a mapped byte buffer will not
 * change the buffer's content and will cause an unspecified exception to be
 * thrown either at the time of the access or at some later time.  It is
 * therefore strongly recommended that appropriate precautions be taken to
 * avoid the manipulation of a mapped file by this program, or by a
 * concurrently running program, except to read or write the file's content.
 *
 * <p> Mapped byte buffers otherwise behave no differently than ordinary direct
 * byte buffers. </p>
 *
 *
 * @author Mark Reinhold
 * @author JSR-51 Expert Group
 * @since 1.4
 */
public abstract class MappedByteBuffer
    extends ByteBuffer
{
```

### 7、Scattering（散射-分散读入） 和 Gathering（聚集-聚集写出(到channel)）

**Scattering**：

​	channel 就把数据 scatter 到多个 buffer 中去。在将数据写入到 buffer 中时，可以采用 buffer 数组，依次写入，一个 buffer 满了就写下一个。将来自于一个 Channel 的数据分散到多个 Buffer 当中,一个满了就用下一个,可以实现数据的分门别类。这样就省去了解析的时间，比如一个消息有三个部分，第一部分是头信息，第二部分是协议信息，第三部分是消息体。

![img](https://upload-images.jianshu.io/upload_images/13084796-38978ddd284411b7.png?imageMogr2/auto-orient/strip|imageView2/2/w/357/format/webp)

**Gatering**：

​	向一个 channel 中进行 gathering write 也是一个写操作，它会把来自多个 buffer 中的数据写入到一个单一的 channel 中。采用 buffer 数组依次写出，一个 buffer 写出完了就对另一个继续写出。

![img](https://upload-images.jianshu.io/upload_images/13084796-31d0cd2f0e3446e9.png?imageMogr2/auto-orient/strip|imageView2/2/w/351/format/webp)

以一个服务端程序为例（**Scattering**）：

```java
public static void main(String[] args) throws IOException {
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.socket().bind(new InetSocketAddress(8899));

    int messageLength = 2 + 3 + 4;
    ByteBuffer[] byteBuffers = new ByteBuffer[3];
    byteBuffers[0] = ByteBuffer.allocate(2);
    byteBuffers[1] = ByteBuffer.allocate(3);
    byteBuffers[2] = ByteBuffer.allocate(4);

    SocketChannel socketChannel = serverSocketChannel.accept();
    while (true) {
        int byteRead = 0;
        while (byteRead < messageLength) {
            long r = socketChannel.read(byteBuffers);
            byteRead += r;
            System.out.println("byteRead: " + byteRead);
            Arrays.asList(byteBuffers).stream()
                    .map(buffer -> "position: " + buffer.position() + ", limit: " + buffer.limit())
                    .forEach(System.out::println);
        }

        Arrays.asList(byteBuffers).forEach(buffer -> buffer.flip());

        long byteWritten = 0;
        while (byteWritten < messageLength) {
            // 把从客户端读取到的内容写会到客户端
            long w = socketChannel.write(byteBuffers);
            byteWritten += w;
        }
        Arrays.asList(byteBuffers).forEach(buffer -> buffer.clear());

        System.out.println("byteRead: " + byteRead + ", byteWritten: "
                + byteWritten + ", messageLength" + messageLength);
    }
}
```

​	启动服务端，通过客户端连接后进行测试，输入9个字节的情况如下，服务端会立即返回，因为已经读取完了9个字节：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525003212801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

​	再进行测试时，不一次性写9个：

![1590338443859](C:\Users\sanhu\AppData\Roaming\Typora\typora-user-images\1590338443859.png)

