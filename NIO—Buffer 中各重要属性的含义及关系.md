## NIO—Buffer 中各重要属性的含义及关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524010526176.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

先看一个 Buffer 类 IntBuffer：

```java
public abstract class IntBuffer extends Buffer implements Comparable<IntBuffer> {
...
```

​	可知，IntBuffer 继承了 Buffer 类。Buffer 类之下还有除 Boolean 之外的 7 种子类，如 IntBuffer 就是其中的一类。

#### 1、接着看一下 Buffer 的源代码阅读：

```java
/**
Buffer 本身是一种包装特定基础数据类型的容器。
    -------------
  1、capacity, limit, position 基础属性：
 	Buffer 的 capacity 属性指的是容器中的元素的个数，其值不可能为负数其永远不会发生变
 	化，分配之后就不会发生变化 
    Buffer 的 limit 属性指的是不能再读写的第一个元素的索引，其值永远不会为负值，且不会
    大于 capacity 属性的值
    Buffer 的 position 属性指的是下一个用来读写的元素的索引（下标），其值永远不会为负
    值，且不会大于 limit 属性的值
    -------------
 2、Buffer 的每个子类都有两类 get 和 put 的操作：
 	相对操作：没当读写一个或多个元素的时候都会进行 position 值的递增操作，如果超出了
 	limit 的值后再进行 get 或 put 操作会抛出一个 BufferUnderflowException 或
    BufferOverflowException 异常。
    相对操作：直接根据指定的索引来 get 或 put 一个元素，且并不会改变 position 的值。如
    果操作下标超过了 limit 的值则都会抛出一个 IndexOutOfBoundsException 异常。
 	
 	可以基于相对 position 的位置通过一个 channel 可以进行数据到 buffer 中传入与传
 	出。
 	-------------
  3、Marking and resetting：
    当调用 reset 方法时，buffer 的 mark 属性会被重置。mark 属性并一定要被定义，一旦定
    义之后就永远不可能为负数，且不会 position 的值。如果定义了 mark 属性的值，且当
    position 或 limit 属性值发生改变后且小于了 mark 的值后，mark 的值则会被丢弃。如
    果在没有定义 mark 属性值的情况下进行了 reset 操作，会抛出 InvalidMarkException
    异常。
    -------------
  4、不变性：
    mark, position, limit, capacity 四者之间的大小：
    mark <= position <= limit <= capacity
    
    新创建了一个 buffer 时，position 的值为 0，mark 属性值未被定义，
    limit 值为 0 或其他值这要取决于 buffer 类型及创建的方式。新创建的 buffer 的每一个
    元素都为 0。
    -------------
  5、Clearing, flipping, and rewinding
    clear：将 limit 的值设置为 capacity、position 设置为 0。也就是将 buffer 恢复
    到一个初始化的状态。
    flip： 将 limit 的值设置为当前 position 的值、position 的值设置为 0.
    rewind：允许重新读取已经包含的数据，也就是不改变 limit 的值，但是将 position 的值
    设置为 0。
    -------------
  6、只读缓存：Read-only buffers
    每一个 buffer 都是可读的，但并不是每一个 buffer 都是可写的，每个 buffer 的可变方
    法（修改上述属性的的值的方法）都是可选的操作，如果在一个只读 buffer 上调用了可变的方
    法会抛一个 ReadOnlyBufferException 异常。
    只读 buffer 不能改变已有元素内容，但是 mark, position, limit 属性的值可以发生变
    化，isReadOnly 可以判断其是否是只读状态。
    -------------
  7、线程安全：
 	buffer 并不是一个线程安全的类，如果存在并发场景，则需要控制并发访问操作。
 	-------------
  8、调用链：
 	在不需要返回特定的值的情况的方法会对象本身，如此便可以将方法的调用链接起来，比如
 	b.flip();
 	b.position(23);
 	b.limit(42);
 	可以使用一下的方式：
 	b.flip().position(23).limit(42);

 * @author Mark Reinhold
 * @author JSR-51 Expert Group
 * @since 1.4
 */
public abstract class Buffer {
    static final int SPLITERATOR_CHARACTERISTICS =
        Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    // Used only by direct buffers
    // NOTE: hoisted here for speed in JNI GetDirectBufferAddress
    long address;

    // Creates a new buffer with the given mark, position, limit, and capacity
    Buffer(int mark, int pos, int lim, int cap) {// package-private
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos)
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }
    ...
}

```

看一个例子：

```java
	public static void main(String[] args) {
        IntBuffer buffer = IntBuffer.allocate(10);

        System.out.printf("capacity: " + buffer.capacity());
        for (int i = 0; i < 5; i++) {
            int randomNumber = new Random().nextInt();
            buffer.put(randomNumber);
        }

        System.out.printf("before flip, limit: " + buffer.limit());

        // 转换操作方式之前一定要进行 flip 操作
        buffer.flip();

        System.out.printf("before flip, limit: " + buffer.limit());
        while (buffer.hasRemaining()) {
            System.out.printf("position: " + buffer.position());
            System.out.printf("limit: " + buffer.limit());
            System.out.printf("capacity: " + buffer.capacity());
            System.out.println(buffer.get());
        }
    }

/**打印如下：
    capacity: 10
    before flip, limit: 10
    before flip, limit: 5
    
    position: 0
    limit: 5
    capacity: 10
    1797159649
    
    position: 1
    limit: 5
    capacity: 10
    1110837799
    
    position: 2
    limit: 5
    capacity: 10
    483517926
    
    position: 3
    limit: 5
    capacity: 10
    1178222960
    
    position: 4
    limit: 5
    capacity: 10
    1586463887
*/
```

#### 2、IntBuffer.allocate(10) 源码如下：

```java
	// 创建一个 position = 0，limit = capacity，mark 未被定义的 IntBuffer,其元素都为 0，它本身有一个数组的属性，且偏移量为 0
	public static IntBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        // 这里实际创建的是 HeapIntBuffer 对象
        return new HeapIntBuffer(capacity, capacity);
    }
```

#### 3、new HeapIntBuffer(capacity, capacity) 源码如下：

```java
// 未定义访问权限符，表示只能在包内调用，且继承了 IntBuffer
class HeapIntBuffer extends IntBuffer {

    HeapIntBuffer(int cap, int lim) { // package-private
		// 调用了父类 IntBuffer 的构造函数
        super(-1, 0, lim, cap, new int[cap], 0);
    }
    ...
}
```

#### 4、IntBuffer 的构造函数如下：

```java
public abstract class IntBuffer extends Buffer implements Comparable<IntBuffer> {

    final int[] hb;                  // Non-null only for heap buffers
    final int offset;
    boolean isReadOnly;                 // Valid only for heap buffers

    IntBuffer(int mark, int pos, int lim, int cap,   // package-private
                 int[] hb, int offset) {
        // 这里可以看到调用了父类 Buffer 的构造函数，父类的构造函数在上边已贴出
        super(mark, pos, lim, cap);
        // 初始化了内部容器
        this.hb = hb;
        this.offset = offset;
    }
```

#### 5、buffer.put(randomNumber) 源码如下：

```
public abstract IntBuffer put(int i);// Buffer 中定义的抽象方法
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524005421522.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

上边知道创建 IntBuffer 时，实际创建的是 HeapIntBuffer，所以看 HeapIntBuffer 中对该抽象方法的实现：

```java
public IntBuffer put(int x) {
    // 这里先对 position 的值进行操作
    hb[ix(nextPutIndex())] = x;
    return this;
}

final int nextPutIndex() {                          // package-private
    if (position >= limit)
        throw new BufferOverflowException();
    return position++;
}

protected int ix(int i) {
    return i + offset;
}
```

#### 6、flip 函数源码如下（Buffer 类中定义）：

```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

#### 7、clear 函数源码如下（Buffer 类中定义）：

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

#### 8、rewind 函数源码如下（Buffer 类中定义）：

```java
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```

#### 9、reset 函数源码如下（Buffer 类中定义）：

```java
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
```

#### 10、mark 函数源码如下（Buffer 类中定义）：

```java
public final Buffer mark() {
    mark = position;
    return this;
}
```

#### 11、其他函数

```java
public final int capacity() {
    return capacity;
}

public final int position() {
    return position;
}

public final Buffer position(int newPosition) {
    if ((newPosition > limit) || (newPosition < 0))
        throw new IllegalArgumentException();
    position = newPosition;
    if (mark > position) mark = -1;
    return this;
}

public final int limit() {
    return limit;
}

public final Buffer limit(int newLimit) {
    if ((newLimit > capacity) || (newLimit < 0))
        throw new IllegalArgumentException();
    limit = newLimit;
    if (position > limit) position = limit;
    if (mark > limit) mark = -1;
    return this;
}
```

