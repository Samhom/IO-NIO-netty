# Disruptor的介绍

## Disruptor简介

- Disruptor是英国外汇交易公司LMAX开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题（在性能测试中发现竟然与I/O操作处于同样的数量级）。
- 基于Disruptor开发的系统单线程能支撑每秒600万订单，2010年在QCon演讲后，获得了业界关注。
- 目前，包括Apache Storm、Camel、Log4j 2在内的很多知名项目都应用了Disruptor以获取高性能。这里所说的队列是系统内部的内存队列，而不是Kafka这样的分布式队列。
- Disruptor实现了队列的功能并且是一个有界队列，可以用于生产者-消费者模型。

## 为什么要有Disruptor，juc下队列存在的问题

- juc下的队列大部分采用加ReentrantLock锁方式保证线程安全。在稳定性要求特别高的系统中，为了防止生产者速度过快，导致内存溢出，只能选择有界队列。
- 加锁的方式通常会严重影响性能。线程会因为竞争不到锁而被挂起，等待其他线程释放锁而唤醒，这个过程存在很大的开销，而且存在死锁的隐患。
- 有界队列通常采用数组实现。但是采用数组实现又会引发另外一个问题false sharing(伪共享)。

## Disruptor如何解决上述问题（设计精髓！）

- **环形数组结构**：为了避免垃圾回收，采用数组而非链表。同时，数组对处理器的缓存机制更加友好（空间局部性原理）。
- **元素位置定位**：数组长度2^n，通过位运算，加快定位的速度。下标采取递增的形式。不用担心index溢出的问题。index是long类型，即使100万QPS的处理速度，也需要30万年才能用完。
- **无锁设计**：每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据。
- **利用缓存行填充解决了伪共享的问题。**
- **实现了基于事件驱动的生产者消费者模型（观察者模式）**：消费者时刻关注着队列里有没有消息，一旦有新消息产生，消费者线程就会立刻把它消费。

## Disruptor的环形数组覆盖策略

- **BlockingWaitStrategy策略**：常见且默认的等待策略，当这个队列里满了，不执行覆盖，而是阻塞等待。使用ReentrantLock+Condition实现阻塞，最节省cpu，但高并发场景下性能最差。适合CPU资源紧缺，吞吐量和延迟并不重要的场景
- **SleepingWaitStrategy策略**：会在循环中不断等待数据。先进行自旋等待如果不成功，则使用Thread.yield()让出CPU,并最终使用LockSupport.parkNanos(1L)进行线程休眠，以确保不占用太多的CPU资源。因此这个策略会产生比较高的平均延时。典型的应用场景就是异步日志。
- **YieldingWaitStrategy策略**：这个策略用于低延时的场合。消费者线程会不断循环监控缓冲区变化，在循环内部使用Thread.yield()让出CPU给别的线程执行时间。如果需要一个高性能的系统，并且对延时比较有严格的要求，可以考虑这种策略。
- **BusySpinWaitStrategy策略**: 采用死循环，消费者线程会尽最大努力监控缓冲区的变化。对延时非常苛刻的场景使用，cpu核数必须大于消费者线程数量。推荐在线程绑定到固定的CPU的场景下使用

## Disruptor的核心概念

- **RingBuffer（环形缓冲区）**：基于数组的内存级别缓存，是创建sequencer(序号)与定义WaitStrategy(拒绝策略)的入口。
- **Disruptor（总体执行入口）**：对RingBuffer的封装，持有RingBuffer、消费者线程池Executor、消费之集合ConsumerRepository等引用。
- **Sequence（序号分配器）**：对RingBuffer中的元素进行序号标记，通过顺序递增的方式来管理进行交换的数据(事件/Event)，一个Sequence可以跟踪标识某个事件的处理进度，同时还能消除伪共享。
- **Sequencer（数据传输器）**：Sequencer里面包含了Sequence，是Disruptor的核心，Sequencer有两个实现类：SingleProducerSequencer(单生产者实现)、MultiProducerSequencer(多生产者实现)，Sequencer主要作用是实现生产者和消费者之间快速、正确传递数据的并发算法
- **SequenceBarrier（消费者屏障）**：用于控制RingBuffer的Producer和Consumer之间的平衡关系，并且决定了Consumer是否还有可处理的事件的逻辑。
- **WaitStrategy（消费者等待策略）**：决定了消费者如何等待生产者将Event生产进Disruptor，WaitStrategy有多种实现策略
- **Event**：从生产者到消费者过程中所处理的数据单元，Event由使用者自定义。
- **EventHandler**：由用户自定义实现，就是我们写消费者逻辑的地方，代表了Disruptor中的一个消费者的接口。
- **EventProcessor**：这是个事件处理器接口，实现了Runnable，处理主要事件循环，处理Event，拥有消费者的Sequence

![Disruptor调用流程图.png](https://zfcq-1256588165.cos.ap-shanghai.myqcloud.com/zfcq-file/2022-02-04/1643962595625/939192727633920000bfc43b6eb758493f88015459edc202e9/939192727633920001.file)

## Disruptor的依赖信息

```xml
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.4.2</version>
</dependency>
```

## Disruptor的使用方式：单个生产者

```java
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Executors;

import com.lmax.disruptor.EventFactory;
import com.lmax.disruptor.EventHandler;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.YieldingWaitStrategy;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

import lombok.Data;

public class DisruptorTest {

	public static void main(String[] args) throws ExecutionException, InterruptedException {

		// 创建disruptor-单生产者
		Disruptor<DisruptorTestEvent> disruptor = new Disruptor<>(new DisruptorTestFactory(), 1024 * 1024,
				Executors.defaultThreadFactory(), ProducerType.SINGLE, // 单生产者
				new YieldingWaitStrategy() // 等待策略
		);

		// 设置消费者用于处理RingBuffer的事件
		disruptor.handleEventsWith(new DisruptorTestEventHandler());
		// 多消费者模式：重复性消费
		// disruptor.handleEventsWith(new DisruptorTestEventHandler2());
		// 消费者开启
		disruptor.start();

		// 创建ringbuffer容器
		RingBuffer<DisruptorTestEvent> ringBuffer = disruptor.getRingBuffer();
		// 创建生产者
		DisruptorTestProducer eventProducer = new DisruptorTestProducer(ringBuffer);
		// 发送消息
		for (int i = 0; i < 100; i++) {
			eventProducer.onData(i, "zhangwei" + i);
		}
// 多个生产者
//        // 创建ringbuffer容器
//        RingBuffer<DisruptorTestEvent> ringBuffer = disruptor.getRingBuffer();
//        // 创建生产者
//        new Thread(()->{
//            //创建生产者
//            DisruptorTestProducer disruptorTestProducer = new DisruptorTestProducer(ringBuffer);
//            // 发送消息
//            for(int i=0;i<100;i++){
//                disruptorTestProducer.onData(i,"zhang"+i);
//            }
//        },"producer1").start();
//
//        new Thread(()->{
//            //创建生产者
//            DisruptorTestProducer disruptorTestProducer = new DisruptorTestProducer(ringBuffer);
//            // 发送消息
//            for(int i=0;i<100;i++){
//                disruptorTestProducer.onData(i,"wei"+i);
//            }
//        },"producer2").start();

		disruptor.shutdown();
	}
}

/**
 * 消息载体类
 */
@Data
class DisruptorTestEvent {
	private long value;
	private String name;
}

/**
 * 定义事件的工厂
 */
class DisruptorTestFactory implements EventFactory<DisruptorTestEvent> {

	@Override
	public DisruptorTestEvent newInstance() {
		return new DisruptorTestEvent();
	}
}

/**
 * 消息（事件）生产者
 */
class DisruptorTestProducer {
	// 事件队列
	private RingBuffer<DisruptorTestEvent> ringBuffer;

	// 构造方法
	public DisruptorTestProducer(RingBuffer<DisruptorTestEvent> ringBuffer) {
		this.ringBuffer = ringBuffer;
	}

	/**
	 * 一个生产的方法
	 */
	public void onData(long value, String name) {
		// 获取事件队列 的下一个槽
		long sequence = ringBuffer.next();
		try {
			// 获取消息（事件）
			DisruptorTestEvent disruptorTestEvent = ringBuffer.get(sequence);
			// 写入消息数据
			disruptorTestEvent.setValue(value);
			disruptorTestEvent.setName(name);
		} catch (Exception e) {
			// TODO 异常处理
			e.printStackTrace();
		} finally {
			System.out.println("生产者发送数据value:" + value + ",name:" + name);
			// 发布事件
			ringBuffer.publish(sequence);
		}
	}
}

/**
 * 消息（事件）消费者
 */
class DisruptorTestEventHandler implements EventHandler<DisruptorTestEvent> {

	@Override
	public void onEvent(DisruptorTestEvent event, long sequence, boolean endOfBatch) throws Exception {
		// TODO 消费逻辑
		System.out.println("消费者获取数据value:" + event.getValue() + ",name:" + event.getName());
	}
}

/**
 * 消息（事件）消费者，第二个
 */
class DisruptorTestEventHandler2 implements EventHandler<DisruptorTestEvent> {

	@Override
	public void onEvent(DisruptorTestEvent event, long sequence, boolean endOfBatch) throws Exception {
		// TODO 消费逻辑
		System.out.println("消费者2获取数据value:" + event.getValue() + ",name:" + event.getName());
	}
}
```