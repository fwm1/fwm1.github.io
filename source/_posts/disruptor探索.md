---
title: disruptor探索
date: 2021-10-30 16:33:03
tags: disruptor
categories: 多线程
---

## 从Logger4j2引出Disruptor

最近把日志框架从logback切换到了log4j2，号称在多线程环境下性能优于log4j和logback数十倍，下面是log4j2官网提供的经典峰值吞吐量对比图。Log4j 2的 Async Loggers 支持使用无锁数据结构（Disruptor），而 Logback、 Log4j 则只支持 ArrayBlockingQueue，多线程下无法避免锁争用，从而导致吞吐量大大降低。
![image-20211031110132988](https://i.loli.net/2021/10/31/HzdbZD95w4CFip1.png)



## Disruptor是什么

> Disruptor是一个高性能的无锁并发框架，提供了一种线程消息传递的方式。

## 为什么有Disruptor

介绍Disruptor之前，我们先来看一看常用的线程安全的内置队列有什么问题。

| 队列                  | 有界性             | 锁结构      | 数据结构   |
| --------------------- | ------------------ | ----------- | ---------- |
| ArrayBlockingQueue    | bounded            | 加锁        | arraylist  |
| LinkedBlockingQueue   | optionally-bounded | 加锁        | linkedlist |
| ConcurrentLinkedQueue | unbounded          | 无锁（CAS） | linkedlist |
| LinkedTransferQueue   | unbounded          | 无锁（CAS） | linkedlist |
| PriorityBlockingQueue | unbounded          | 加锁        | heap       |
| DelayQueue            | unbounded          | 加锁        | heap       |

### ArrayBlockingQueue的缺点

#### 加锁性能损耗

#### 伪共享

######  什么是伪共享

首先了解计算机CPU缓存基本结构。L1缓存、L2缓存、L3缓存、主存，容量依次增大，访问速度依次减小。CPU读取主存中的数据会比从L1中读取慢2个数量级。

| 从CPU到                                  | **大约需要的CPU周期** | **大约需要的时间** |
| ---------------------------------------- | --------------------- | ------------------ |
| 主存                                     | -                     | 约60-80ns          |
| QPI 总线传输(between sockets, not drawn) | -                     | 约20ns             |
| L3 cache                                 | 约40-45 cycles        | 约15ns             |
| L2 cache                                 | 约10 cycles           | 约3ns              |
| L1 cache                                 | 约3-4 cycles          | 约1ns              |
| 寄存器                                   | 1 cycle               | -                  |

###### 缓存行

Cache是由很多个cache line组成的。每个cache line通常是64字节，并且它有效地引用主内存中的一块儿地址。一个Java的long类型变量是8字节，因此在一个缓存行中可以存8个long类型的变量。CPU每次从主存中拉取数据时，会把相邻的数据也存入同一个cache line。在访问一个long数组的时候，如果数组中的一个值被加载到缓存中，它会自动加载另外7个。因此你能非常快的遍历这个数组。

ArrayBlockingQueue有三个成员变量：

- takeIndex：需要被取走的元素下标 
-  putIndex：可被元素插入的位置的下标 
-  count：队列中元素的数量

这三个变量很容易放到一个缓存行中，但是之间修改没有太多的关联。**所以每次修改，都会使之前缓存的数据失效**，从而不能完全达到共享的效果。当生产者线程put一个元素到ArrayBlockingQueue时，putIndex会修改，从而导致消费者线程的缓存中的缓存行无效，需要从主存中重新读取。**这种无法充分使用缓存行特性的现象，称为伪共享**。

对于伪共享，一般的解决方案是，**增大数组元素的间隔使得由不同线程存取的元素位于不同的缓存行上**，以空间换时间。

## Disruptor的设计方案

- 环形数组。采用数组而非链表。数组对处理器的缓存机制更加友好
- 元素定位采用位运算。数组长度2^n，通过位运算，加快定位的速度。下标采取递增的形式。不用担心index溢出的问题。index是long类型，即使100万QPS的处理速度，也需要30万年才能用完。
- 无锁设计。每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据。

#### 单个生产者

1. 申请写入m个元素；
2. 若是有m个元素可以入，则返回最大的序列号。这儿主要判断是否会覆盖未读的元素；
3. 若是返回的正确，则生产者开始写入元素。

<img src="https://i.loli.net/2021/11/27/2g9kZhAzeI8JNG1.png" alt="image-20211127161425403" style="zoom: 67%;" />

<img src="https://i.loli.net/2021/11/27/qXOAkbCV5EScghT.png" alt="image-20211127161506809" style="zoom:67%;" />

```java
//com.lmax.disruptor.SingleProducerSequencer#next(int)
//获取下一个可用的index
@Override
public long next(int n)
{
    if (n < 1)
    {
        throw new IllegalArgumentException("n must be > 0");
    }

    long nextValue = this.nextValue;

    long nextSequence = nextValue + n;
    long wrapPoint = nextSequence - bufferSize;
    long cachedGatingSequence = this.cachedValue;

    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
    {
        cursor.setVolatile(nextValue);  // StoreLoad fence

        long minSequence;
        while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
        {
            LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?
        }

        this.cachedValue = minSequence;
    }

    this.nextValue = nextSequence;

    return nextSequence;
}
```

#### 多个生产者

多个生产者的情况下，会遇到“如何防止多个线程重复写同一个元素”的问题。Disruptor的解决方法是，**每个线程获取不同的一段数组空间进行操作**。这个通过CAS很容易达到。**只需要在分配元素的时候，通过CAS判断一下这段空间是否已经分配出去即可**。

但是会遇到一个新问题：如何防止读取的时候，读到还未写的元素。Disruptor在多个生产者的情况下，引入了一个与Ring Buffer大小相同的buffer：**available Buffer**。当某个位置写入成功的时候，便把availble Buffer相应的位置置位，标记为写入成功。**读取的时候，会遍历available Buffer，来判断元素是否已经就绪**。

##### 读数据

1. 申请读取到序号n；
2. 若writer cursor >= n，这时仍然无法确定连续可读的最大下标。从reader cursor开始读取available Buffer，一直查到第一个不可用的元素，然后返回最大连续可读元素的位置；
3. 消费者读取元素。

如下图所示，读线程读到下标为2的元素，三个线程Writer1/Writer2/Writer3正在向RingBuffer相应位置写数据，写线程被分配到的最大元素下标是11。

读线程申请读取到下标从3到11的元素，判断writer cursor>=11。然后开始读取availableBuffer，从3开始，往后读取，发现下标为7的元素没有生产成功，于是WaitFor(11)返回6。因此，消费者读取下标从3到6共计4个元素。

![image-20211127162333405](https://i.loli.net/2021/11/27/R3TkwUDjHZsA8dn.png)

##### 写数据

1. 申请写入m个元素；
2. 若是有m个元素可以写入，则返回最大的序列号。每个生产者会被分配一段独享的空间；
3. 生产者写入元素，写入元素的同时设置available Buffer里面相应的位置，以标记自己哪些位置是已经写入成功的。

如下图所示，Writer1和Writer2两个线程写入数组，都申请可写的数组空间。Writer1被分配了下标3到下表5的空间，Writer2被分配了下标6到下标9的空间。

Writer1写入下标3位置的元素，同时把available Buffer相应位置置位，标记已经写入成功，往后移一位，开始写下标4位置的元素。Writer2同样的方式。最终都写入完成。

![image-20211127162555454](https://i.loli.net/2021/11/27/X8l3DmSkHco1uA6.png)

```java
@Override
public long next(int n)
{
    if (n < 1)
    {
        throw new IllegalArgumentException("n must be > 0");
    }

    long current;
    long next;

    do
    {
        current = cursor.get();
        next = current + n;

        long wrapPoint = next - bufferSize;
        long cachedGatingSequence = gatingSequenceCache.get();

        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
        {
            long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

            if (wrapPoint > gatingSequence)
            {
                LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                continue;
            }

            gatingSequenceCache.set(gatingSequence);
        }
        else if (cursor.compareAndSet(current, next))
        {
            break;
        }
    }
    while (true);

    return next;
}
```



## Disruptor源码初探

![image-20211127181001523](https://i.loli.net/2021/11/27/1mvxdBl67eziOcI.png)

### 相关类和接口

- **Sequenced** 环形缓存区与序号相关的操作。
  - int getBufferSize() 获取缓存区大小
  - boolean hasAvailableCapacity(int requiredCapacity) 是否还有requiredCapacity个空间供写入线程写入数据
  - long remainingCapacity() 当前剩余容量
  - long next() 获取下一个可写入的下标
  - long next(int n) 从环形队列中获取n个可用的下标，返回值为这批最大的可写入序号
  - long tryNext() throws InsufficientCapacityException 尝试从环形队列中获取一个可写入位置，如果没有空闲位置供写入则抛出异常
  - long tryNext(int n) throws InsufficientCapacityException 尝试从环形队列中获取n个可写入位置，如果没有空闲位置供写入则抛出异常
  - void publish(long sequence) 将处于下标sequence的位置“发布”，此时消费者可以从环形队列中消费
  - void publish(long lo, long hi) 将从 lo 到 hi 的下标之间的数据发布
- **DataProvider** 数据提供者，只提供了根据下标位置获取数据
- **Cursored **游标，当前处理的下标
- **EventSink **数据 sink，主要是提供了丰富的publish方法
- **RingBufferPad**、**RingBufferFields** 填充实现，主要解决“伪共享”
- **RingBuffer** 环形缓存区实现类

### RingBuffer解决伪共享

在RingBuffer中环形队列在底层需要维护一个存储数组，那么要保证数组中的每个下标都不要和其他无关变量缓存到同一个cache line中，通常的办法是通常的方案是在前后数组填充128个字节（理论上只要64字节？）。

```java
static
{
    //返回当前jvm中用来表示一个数组下标占用的字节数，64位操作系统开启了指针压缩将返回4，否则返回8，默认开启了指针压缩。
    final int scale = UNSAFE.arrayIndexScale(Object[].class);
    if (4 == scale)
    {
        REF_ELEMENT_SHIFT = 2;
    }
    else if (8 == scale)
    {
        REF_ELEMENT_SHIFT = 3;
    }
    else
    {
        throw new IllegalStateException("Unknown pointer size");
    }
    BUFFER_PAD = 128 / scale;
    // Including the buffer pad in the array base offset
    // UNSAFE.arrayBaseOffset可以获取数组的起始位置
    REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
}
```

![image-20211127170539836](https://i.loli.net/2021/11/27/Bqj1v5TpzWrJIyF.png)



### 无锁化实现

Disruptor写数据模板

```java
//获取下一个可写的下标
long sequence = ringBuffer.next();
try {
    //从ringbuffer中拿出此下标对应的
	Event e = ringBuffer.get(sequence);
	// Do some work with the event.
} finally {
	ringBuffer.publish(sequence);
}
```

RingBuffer#next()直接调用sequencer#next()

#### Sequencer

**MultiProducerSequencer**、**SingleProducerSequencer**两个具体的实现，也是Disruptor实现无锁化的核心要点。

![image-20211127172622598](https://i.loli.net/2021/11/27/mTvM4R1bZJ9gB8P.png)

#### 环形队列写数据原理

环形队列，就是对数组进行重复利用，当writeIndex移动到队列最后一个位置再+1时，会移动到队列开头重新写入，因此会有数据覆盖的问题。如果队列开头的元素暂未被读取，那么直接覆盖掉就丢失数据了。为了避免这种情况，需要增加对writeIndex和readIndex约束，readIndex表示第一条待读取的位置：
$$
writeIndex - readIndex < bufferSize
$$
知道这个条件后再继续看**MultiProducerSequencer#next**方法

```java
    /**
    * @see Sequencer#next(int)
    */
    @Override
    public long next(int n)
    {
        if (n < 1)
        {
            throw new IllegalArgumentException("n must be > 0");
        }

        long current;
        long next;

        do
        {
            //环形队列已使用的最大序号，下一个可写序号从 cursor + 1 开始
            current = cursor.get();
            next = current + n;

            //
            long wrapPoint = next - bufferSize;
            //消费端已处理的最小序列号，即环形队列中的readIndex
            long cachedGatingSequence = gatingSequenceCache.get();

            //如果暂时不能写入序号
            if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
            {
                long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

                if (wrapPoint > gatingSequence)
                {
                    LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                    continue;
                }

                gatingSequenceCache.set(gatingSequence);
            }
            else if (cursor.compareAndSet(current, next))
            {
                //使用CAS命令尝试更新，如果更新成功，则返回next给Producer
                break;
            }
        }
        while (true);

        return next;
    }
```

#### 多个Consumer读数据

在disruptor中，并发消费的实现类有**WorkerPool**、**BatchEventProcessor**

##### WorkerPool 

WorkerPool中**每一个线程一次只处理一个序号**

WorkerPool中有多个WorkProcessor，具体看WorkProcessor#run方法

- nextSequence 每一个WorkerPool维护一个整体的处理进度序号，会被多个WorkProcessor共同竞争获取
- sequence 每一个WorkProcessor内部会维护一个当前已处理的序号

```java
	@Override
    public void run()
    {
        if (!running.compareAndSet(false, true))
        {
            throw new IllegalStateException("Thread is already running");
        }
        sequenceBarrier.clearAlert();

        notifyStart();

        boolean processedSequence = true;
        long cachedAvailableSequence = Long.MIN_VALUE;
        long nextSequence = sequence.get();
        T event = null;
        while (true)
        {
            try
            {
                // if previous sequence was processed - fetch the next sequence and set
                // that we have successfully processed the previous sequence
                // typically, this will be true
                // this prevents the sequence getting too far forward if an exception
                // is thrown from the WorkHandler
                if (processedSequence)
                {
                    processedSequence = false;
                    // (1) 通过while+CAS获取整个WorkerPool的下一个待处理的下标nextSequence
                    do
                    {
                        nextSequence = workSequence.get() + 1L;
                        sequence.set(nextSequence - 1L);
                    }
                    while (!workSequence.compareAndSet(nextSequence - 1L, nextSequence));
                }
				// (2)如果当前可处理的序号大于等于nextSequence，即可处理该序号中的数据，
                // 否则通过barrier的waitFor等待待处理序号可用，即等待Producer发布该序号
                if (cachedAvailableSequence >= nextSequence)
                {
                    event = ringBuffer.get(nextSequence);
                    workHandler.onEvent(event);
                    processedSequence = true;
                }
                else
                {
                    cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence);
                }
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (!running.get())
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                // handle, mark as processed, unless the exception handler threw an exception
                exceptionHandler.handleEventException(ex, nextSequence, event);
                processedSequence = true;
            }
        }

        notifyShutdown();

        running.set(false);
    }
```



##### BatchEventProcessor

批处理实现，一次获取多个序号。

```java
private void processEvents()
    {
        T event = null;
    	// 获取下一个可处理序号
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {	//通过sequenceBarrier获取可用的序号availableSequence
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }
				//一次性处理多个序号，直到达到availableSequence
                while (nextSequence <= availableSequence)
                {
                    event = dataProvider.get(nextSequence);
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }
				//设置当前已经处理的序号
                sequence.set(availableSequence);
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (running.get() != RUNNING)
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }
```

