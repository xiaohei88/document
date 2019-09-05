## 7.1 Disruptor

#### 7.1.1 Disruptor是什么

> [!COMMENT]
>
> Disruptor是一个高性能的异步处理框架，或者可以认为是线程通信的高效低延时的内存消息组件，它最大特点是高性能，其LAMX架构可以获得美妙6百万订单，用1微妙的延迟获得吞吐量为100K+。

#### 7.1.2 JDK内置内存队列？

| 队列                  | 加锁方式 | 是否有节 |  数据结构  |
| :-------------------- | :------: | :------: | :--------: |
| ArrayBlockingQueue    |   加锁   |   有界   | ArrayList  |
| LinkedBlockingQueue   |   加锁   |   无界   | LinkedList |
| ConcurrentLinkedQueue |   CAS    |   无界   | LinkedList |
| LinkedTransferQueue   |   CAS    |   无界   | LinkedList |

> 我们知道CAS算法比通过加锁实现同步性能高很多，而上表可以看出基于CAS实现的队列都是无界的，而有界队列是通过同步实现的。在系统稳定性要求比较高的场景下，为了防止生产者速度过快，如果采用无界队列会最终导致内存溢出，只能选择有界队列。而有界队列只有`ArrayBlockingQueue`，该队列是通过加锁实现的，在请求锁和释放锁时对性能开销很大，这时候基于有界队列的高性能的Disruptor就应运而生。

#### 7.1.3 Disruptor如何实现高性能

>  Disruptor实现高性能主要体现了去掉了锁，采用CAS算法，同时内部通过环形队列实现有界队列。

- 环形数据结构
  为了避免垃圾回收，采用数组而非链表。同时，数组对处理器的缓存机制更加友好。
  - 初始化创建`RingBuffer.RingBufferFields.entries`数组大小为`ringBufferSize`
- 元素位置定位
  数组长度2^n，通过位运算，加快定位的速度。下标采取递增的形式。不用担心index溢出的问题。index是long类型，即使100万QPS的处理速度，也需要30万年才能用完。
  - 通过`Sequence`来定义索引
  - 对`Sequence`获取采用`LockSupport.parkNanos`
- 无锁设计
  每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据。整个过程通过原子变量CAS，保证操作的线程安全。
  - 生产者和消费者采用`ReentrantLock`

#### 7.1.4 Disruptor可以用来做什么？

>  当前业界开源组件使用Disruptor的包括Log4j2、Apache Storm等，它可以用来作为高性能的有界内存队列，基于生产者消费者模式，实现一个/多个生产者对应多个消费者。它也可以认为是观察者模式的一种实现，或者发布订阅模式。

#### 7.1.5 为什么要使用Disruptor?

> 使用Disruptor，主要用于对性能要求高、延迟低的场景，它通过“榨干”机器的性能来换取处理的高性能。如果你的项目有对性能要求高，对延迟要求低的需求，并且需要一个无锁的有界队列，来实现生产者/消费者模式，那么Disruptor是你的不二选择。

#### 7.1.6 DEMO

```java
package com.henry.learn.disruptor;

import com.lmax.disruptor.BlockingWaitStrategy;
import com.lmax.disruptor.EventFactory;
import com.lmax.disruptor.EventHandler;
import com.lmax.disruptor.EventTranslatorOneArg;
import com.lmax.disruptor.ExceptionHandler;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

import java.util.concurrent.ThreadFactory;

/**
 * @author Henry
 * @date 2019/2/28 6:35 PM
 */
public class DisruptorTest {

    /**
     * 消息事件类
     */
    public static class MessageEvent {
        /**
         * 原始消息
         */
        private String message;

        public String getMessage() {
            return message;
        }

        public void setMessage(String message) {
            this.message = message;
        }
    }

    /**
     * 消息事件工厂类
     */
    public static class MessageEventFactory implements EventFactory<MessageEvent> {

        @Override
        public MessageEvent newInstance() {
            return new MessageEvent();
        }
    }

    /**
     * 消息转换类，负责将消息转换为事件
     */
    public static class MessageEventTranslator implements EventTranslatorOneArg<MessageEvent, String> {
        @Override
        public void translateTo(MessageEvent messageEvent, long l, String s) {
            messageEvent.setMessage(s);
        }
    }

    /**
     * 消费者线程工厂类
     */
    public static class MessageThreadFactory implements ThreadFactory {

        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "Simple Disruptor Test Thread");
        }
    }

    /**
     * 消息事件处理类，这里只打印消息
     */
    public static class MessageEventHandler implements EventHandler<MessageEvent> {
        @Override
        public void onEvent(MessageEvent messageEvent, long l, boolean b) throws Exception {
            System.out.println(messageEvent.getMessage());
        }
    }

    /**
     * 异常处理类
     */
    public static class MessageExceptionHandler implements ExceptionHandler<MessageEvent> {
        @Override
        public void handleEventException(Throwable ex, long sequence, MessageEvent event) {
            ex.printStackTrace();
        }

        @Override
        public void handleOnStartException(Throwable ex) {
            ex.printStackTrace();

        }

        @Override
        public void handleOnShutdownException(Throwable ex) {
            ex.printStackTrace();

        }
    }

    /**
     * 消息生产者类
     */
    public static class MessageEventProducer {
        private RingBuffer<MessageEvent> ringBuffer;

        public MessageEventProducer(RingBuffer<MessageEvent> ringBuffer) {
            this.ringBuffer = ringBuffer;
        }

        /**
         * 将接收到的消息输出到ringBuffer
         *
         * @param message
         */
        public void onData(String message) {
            EventTranslatorOneArg<MessageEvent, String> translator = new MessageEventTranslator();
            ringBuffer.publishEvent(translator, message);
        }
    }

    public static void main(String[] args) {
        String message = "Hello Disruptor!";
        int ringBufferSize = 1024;//必须是2的N次方

        Disruptor<MessageEvent> disruptor = new Disruptor<>(new MessageEventFactory(), ringBufferSize,
                new MessageThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
        disruptor.handleEventsWith(new MessageEventHandler());
        disruptor.setDefaultExceptionHandler(new MessageExceptionHandler());
        RingBuffer<MessageEvent> ringBuffer = disruptor.start();
        MessageEventProducer producer = new MessageEventProducer(ringBuffer);
        producer.onData(message);
    }
}
```