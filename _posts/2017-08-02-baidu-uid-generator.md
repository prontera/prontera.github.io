---
layout:     post
title:      "百度 - UidGenerator源码解析"
subtitle:   "Understand how Baidu UidGenerator works"
author:     "Chris"
header-img: "img/post-bg-8.jpg"
tags:
    - Java
    - 源码研读
---

## 简介

[UidGenerator](https://github.com/baidu/uid-generator)是Java实现的，基于[Snowflake](https://github.com/twitter/snowflake)算法的唯一ID生成器。UidGenerator以组件形式工作在应用项目中，支持自定义workerId位数和初始化策略，从而适用于[Docker](https://www.docker.com/)等虚拟化环境下实例自动重启、漂移等场景。 在实现上，UidGenerator通过采用RingBuffer来缓存已生成的UID，并行化UID的生产和消费，同时对CacheLine补齐，避免了由RingBuffer带来的硬件级「伪共享」问题。

## 源码解析

本文基于commit id `ba696f535ba6b000b96dd73a7b697e4a00c88085`所写，为编写本文时（2017-08-02 21:59:47）的最新的Master分支，阅读时须注意未来的版本迭代有可能造成功能上的差异。

### 目录结构

```shell
 com
 └── baidu
     └── fsg
         └── uid
             ├── BitsAllocator.java			- Bit分配器(C)
             ├── UidGenerator.java			- UID生成的接口(I)
             ├── buffer
             │   ├── BufferPaddingExecutor.java		- 填充RingBuffer的执行器(C)
             │   ├── BufferedUidProvider.java		- RingBuffer中UID的提供者(C)
             │   ├── RejectedPutBufferHandler.java	- 拒绝Put到RingBuffer的处理器(C)
             │   ├── RejectedTakeBufferHandler.java	- 拒绝从RingBuffer中Take的处理器(C)
             │   └── RingBuffer.java			- 内含两个环形数组(C)
             ├── exception
             │   └── UidGenerateException.java		- 运行时异常
             ├── impl
             │   ├── CachedUidGenerator.java		- RingBuffer存储的UID生成器(C)
             │   └── DefaultUidGenerator.java		- 无RingBuffer的默认UID生成器(C)
             ├── utils
             │   ├── DateUtils.java
             │   ├── DockerUtils.java
             │   ├── EnumUtils.java
             │   ├── NamingThreadFactory.java
             │   ├── NetUtils.java
             │   ├── PaddedAtomicLong.java
             │   └── ValuedEnum.java
             └── worker
                 ├── DisposableWorkerIdAssigner.java	- 用完即弃的WorkerId分配器(C)
                 ├── WorkerIdAssigner.java		- WorkerId分配器(I)
                 ├── WorkerNodeType.java		- 工作节点类型(E)
                 ├── dao
                 │   └── WorkerNodeDAO.java		- MyBatis Mapper
                 └── entity
                     └── WorkerNodeEntity.java		- MyBatis Entity
```

### 组件功能简述

UidGenerator在应用中是以Spring组件的形式提供服务，`DefaultUidGenerator`提供了最简单的Snowflake式的生成模式，但是并没有使用任何缓存来预存UID，在需要生成ID的时候即时进行计算。而`CachedUidGenerator`是一个使用RingBuffer预先缓存UID的生成器，在初始化时就会填充整个RingBuffer，并在take()时检测到少于指定的填充阈值之后就会异步地再次填充RingBuffer（默认值为50%），另外可以启动一个定时器周期性检测阈值并及时进行填充。

本文将着重介绍`CachedUidGenerator`及其背后的组件是如何运作的，在此之前我们先了解某些核心类是如何运转。

### BitsAllocator - Bit分配器

整个UID由64bit组成，以下图为例，1bit是符号位，其余63位由`deltaSeconds`、`workerId`和`sequence`组成，注意`sequence`被放在最后，可**方便直接进行求和或自增操作**。

![](https://github.com/baidu/uid-generator/blob/master/doc/snowflake.png?raw=true)

该类主要接收上述3个用于组成UID的元素，并计算出各个元素的最大值和对应的位偏移。其申请UID时的方法如下，由这3个元素进行或操作进行拼接。

```java
    /**
     * Allocate bits for UID according to delta seconds & workerId & sequence<br>
     * <b>Note that: </b>The highest bit will always be 0 for sign
     *
     * @param deltaSeconds
     * @param workerId
     * @param sequence
     * @return
     */
    public long allocate(long deltaSeconds, long workerId, long sequence) {
        return (deltaSeconds << timestampShift) | (workerId << workerIdShift) | sequence;
    }
```

## DisposableWorkerIdAssigner - Worker ID分配器

本类用于为每个工作机器分配一个唯一的ID，目前来说是用完即弃，在初始化Bean的时候会自动向MySQL中插入一条关于该服务的启动信息，待MySQL返回其自增ID之后，使用该ID作为工作机器ID并柔和到UID的生成当中。

```java
    @Transactional
    public long assignWorkerId() {
        // build worker node entity
        WorkerNodeEntity workerNodeEntity = buildWorkerNode();

        // add worker node for new (ignore the same IP + PORT)
        workerNodeDAO.addWorkerNode(workerNodeEntity);
        LOGGER.info("Add worker node:" + workerNodeEntity);

        return workerNodeEntity.getId();
    }
```

`buildWorkerNode()`为获取该启动服务的信息，兼容Docker服务。

## RingBuffer - 用于存储UID的双环形数组结构

我们先看RingBuffer的field outline，这样能大致了解到他的工作模式：

```java
    /**
     * Constants
     */
    private static final int START_POINT = -1;
    private static final long CAN_PUT_FLAG = 0L;
    private static final long CAN_TAKE_FLAG = 1L;
	// 默认扩容阈值
    public static final int DEFAULT_PADDING_PERCENT = 50;

    /**
     * The size of RingBuffer's slots, each slot hold a UID
     * <p>
     * buffer的大小为2^n
     */
    private final int bufferSize;
    /**
     * 因为bufferSize为2^n，indexMask为bufferSize-1，作为被余数可快速取模
     */
    private final long indexMask;
    /**
     * 盛装UID的数组
     */
    private final long[] slots;
    /**
     * 盛装flag的数组(是否可读或者可写)
     */
    private final PaddedAtomicLong[] flags;

    /**
     * Tail: last position sequence to produce
     */
    private final AtomicLong tail = new PaddedAtomicLong(START_POINT);

    /**
     * Cursor: current position sequence to consume
     */
    private final AtomicLong cursor = new PaddedAtomicLong(START_POINT);

    /**
     * Threshold for trigger padding buffer
     */
    private final int paddingThreshold;

    /**
     * Reject putbuffer handle policy
     * <p>
     * 拒绝方式为打印日志
     */
    private RejectedPutBufferHandler rejectedPutHandler = this::discardPutBuffer;
    /**
     * Reject take buffer handle policy
     * <p>
     * 拒绝方式为抛出异常并打印日志
     */
    private RejectedTakeBufferHandler rejectedTakeHandler = this::exceptionRejectedTakeBuffer;

    /**
     * Executor of padding buffer
     * <p>
     * 填充RingBuffer的executor
     */
    private BufferPaddingExecutor bufferPaddingExecutor;
```

RingBuffer内两个环形数组，一个名为`slots`的用于存放UID的long类型数组，另一个名为`flags`的用于存放读写标识的PaddedAtomicLong类型数组。

即使是**不同线程间**对slots进行**串行写操作（下文会详述）**在多核处理器下应该也会使得该数组发生伪共享问题，因为Java线程在目前来说并不能绑定CPU，所以在修改相同的Cache Line的时候，是有十分可能产生RFO信号的。

那为什么一个使用long而另一个使用PaddedAtomicLong呢？

原因是`slots`数组选用原生类型是为了高效地读取，数组在内存中是连续分配的，当你读取第0个元素的之后，后面的若干个数组元素也会同时被加载。分析代码即可发现`slots`实质是属于**多读少写**的变量，所以使用原生类型的收益更高。而`flags`则是会频繁进行写操作，为了避免伪共享问题所以手工进行补齐。如果使用的是JDK8，也可以使用注解`sun.misc.Contended`在类或者字段上声明，在使用JVM参数`-XX:-RestrictContended`时会自动进行补齐。

### RingBuffer的填充操作

我们需要注意的是`put(long)`方法是一个**同步**方法，换句话说就是**串行写**，保证了填充slot和移动tail是原子操作。

```java
    /**
     * Put an UID in the ring & tail moved<br>
     * We use 'synchronized' to guarantee the UID fill in slot & publish new tail sequence as atomic operations<br>
     * <p>
     * <b>Note that: </b> It is recommended to put UID in a serialize way, cause we once batch generate a series UIDs and put
     * the one by one into the buffer, so it is unnecessary put in multi-threads
     *
     * @param uid
     * @return false means that the buffer is full, apply {@link RejectedPutBufferHandler}
     */
    public synchronized boolean put(long uid) {
        long currentTail = tail.get();
        long currentCursor = cursor.get();
        // 首次put时，currentTail为-1，currentCursor为0，此时distance为-1
        long distance = currentTail - (currentCursor == START_POINT ? 0 : currentCursor);
        // tail catches the cursor, means that you can't put anything cause of RingBuffer is full
        if (distance == bufferSize - 1) {
            rejectedPutHandler.rejectPutBuffer(this, uid);
            return false;
        }

        // 1. pre-check whether the flag is CAN_PUT_FLAG
        // 首次put时，currentTail为-1
        int nextTailIndex = calSlotIndex(currentTail + 1);
        if (flags[nextTailIndex].get() != CAN_PUT_FLAG) {
            rejectedPutHandler.rejectPutBuffer(this, uid);
            return false;
        }

        // 2. put UID in the next slot
        slots[nextTailIndex] = uid;
        // 3. update next slot' flag to CAN_TAKE_FLAG
        flags[nextTailIndex].set(CAN_TAKE_FLAG);
        // 4. publish tail with sequence increase by one
        tail.incrementAndGet();

        // The atomicity of operations above, guarantees by 'synchronized'. In another word,
        // the take operation can't consume the UID we just put, until the tail is published(tail.incrementAndGet())
        return true;
    }
```

### RingBuffer的读取操作

UID的读取是一个lock free操作，使用CAS成功将`tail`往后移动之后即视为线程安全。

```java
    /**
     * Take an UID of the ring at the next cursor, this is a lock free operation by using atomic cursor<p>
     * <p>
     * Before getting the UID, we also check whether reach the padding threshold,
     * the padding buffer operation will be triggered in another thread<br>
     * If there is no more available UID to be taken, the specified {@link RejectedTakeBufferHandler} will be applied<br>
     *
     * @return UID
     * @throws IllegalStateException if the cursor moved back
     */
    public long take() {
        // spin get next available cursor
        long currentCursor = cursor.get();
        // cursor初始化为-1，现在cursor等于tail，所以初始化时nextCursor为-1
        long nextCursor = cursor.updateAndGet(old -> old == tail.get() ? old : old + 1);

        // check for safety consideration, it never occurs
        // 初始化或者全部UID耗尽时nextCursor == currentCursor
        Assert.isTrue(nextCursor >= currentCursor, "Curosr can't move back");

        // trigger padding in an async-mode if reach the threshold
        long currentTail = tail.get();
        // 会有多个线程去触发padding事件，但最终只会有一条线程执行padding操作
        if (currentTail - nextCursor < paddingThreshold) {
            LOGGER.info("Reach the padding threshold:{}. tail:{}, cursor:{}, rest:{}", paddingThreshold, currentTail,
                    nextCursor, currentTail - nextCursor);
            bufferPaddingExecutor.asyncPadding();	// (a)
        }

        // cursor catch the tail, means that there is no more available UID to take
        if (nextCursor == currentCursor) {
            rejectedTakeHandler.rejectTakeBuffer(this);
        }

        // 1. check next slot flag is CAN_TAKE_FLAG
        int nextCursorIndex = calSlotIndex(nextCursor);
        // 这个位置必须要是可以TAKE
        Assert.isTrue(flags[nextCursorIndex].get() == CAN_TAKE_FLAG, "Curosr not in can take status");

        // 2. get UID from next slot
        // 取出UID
        long uid = slots[nextCursorIndex];
        // 3. set next slot flag as CAN_PUT_FLAG.
        // 告知flags数组这个位置是可以被重用了
        flags[nextCursorIndex].set(CAN_PUT_FLAG);

        // Note that: Step 2,3 can not swap. If we set flag before get value of slot, the producer may overwrite the
        // slot with a new UID, and this may cause the consumer take the UID twice after walk a round the ring
        return uid;
    }
```

在`(a)`处可以看到当达到默认填充阈值50%时，即slots被消费大于50%的时候进行异步填充，这个填充由`BufferPaddingExecutor`所执行的，下面我们马上看看这个执行者的代码。

## BufferPaddingExecutor - RingBuffer元素填充器

该用于填充RingBuffer的执行者最主要的执行方法如下

```java
    /**
     * Padding buffer fill the slots until to catch the cursor
     * <p>
     * 该方法被即时填充和定期填充所调用
     */
    public void paddingBuffer() {
        LOGGER.info("{} Ready to padding buffer lastSecond:{}. {}", this, lastSecond.get(), ringBuffer);

        // is still running
        // 这个是代表填充executor在执行，不是RingBuffer在执行。为免多个线程同时扩容。
        if (!running.compareAndSet(false, true)) {
            LOGGER.info("Padding buffer is still running. {}", ringBuffer);
            return;
        }

        // fill the rest slots until to catch the cursor
        boolean isFullRingBuffer = false;
        while (!isFullRingBuffer) {
            // 填充完指定SECOND里面的所有UID，直至填满
            List<Long> uidList = uidProvider.provide(lastSecond.incrementAndGet());
            for (Long uid : uidList) {
                isFullRingBuffer = !ringBuffer.put(uid);
                if (isFullRingBuffer) {
                    break;
                }
            }
        }

        // not running now
        running.compareAndSet(true, false);
        LOGGER.info("End to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);
    }
```

当线程池分发多条线程来执行填充任务的时候，成功抢夺运行状态的线程会真正执行对RingBuffer填充，直至全部填满，其他抢夺失败的线程将会直接返回。

1. 该类还提供定时填充功能，如果有设置开关则会生效，默认不会启用周期性填充。

```java
    /**
     * Start executors such as schedule
     */
    public void start() {
        if (bufferPadSchedule != null) {
            bufferPadSchedule.scheduleWithFixedDelay(this::paddingBuffer, scheduleInterval, scheduleInterval, TimeUnit.SECONDS);
        }
    }
```

2. 在take()方法中检测到达到填充阈值时，会进行异步填充。

```java
    /**
     * Padding buffer in the thread pool
     */
    public void asyncPadding() {
        bufferPadExecutors.submit(this::paddingBuffer);
    }
```

## 其他函数式接口

`BufferedUidProvider` - UID的提供者，在本仓库中以lambda形式出现在`com.baidu.fsg.uid.impl.CachedUidGenerator#nextIdsForOneSecond`

`RejectedPutBufferHandler` - 当RingBuffer满时拒绝继续添加的处理者，在本仓库中的表现形式为`com.baidu.fsg.uid.buffer.RingBuffer#discardPutBuffer`

`RejectedTakeBufferHandler` - 当RingBuffer为空时拒绝获取UID的处理者，在本仓库中的表现形式为`com.baidu.fsg.uid.buffer.RingBuffer#exceptionRejectedTakeBuffer`

## CachedUidGenerator - 使用RingBuffer的UID生成器

该类在应用中作为Spring Bean注入到各个组件中，主要作用是初始化`RingBuffer`和`BufferPaddingExecutor`。获取ID是通过委托RingBuffer的take()方法达成的，而最重要的方法为`BufferedUidProvider`的提供者，即lambda表达式中的`nextIdsForOneSecond(long)`方法

```java
    /**
     * Get the UIDs in the same specified second under the max sequence
     *
     * @param currentSecond
     * @return UID list, size of {@link BitsAllocator#getMaxSequence()} + 1
     */
    protected List<Long> nextIdsForOneSecond(long currentSecond) {
        // Initialize result list size of (max sequence + 1)
        int listSize = (int) bitsAllocator.getMaxSequence() + 1;
        List<Long> uidList = new ArrayList<>(listSize);

        // Allocate the first sequence of the second, the others can be calculated with the offset
        long firstSeqUid = bitsAllocator.allocate(currentSecond - epochSeconds, workerId, 0L);
        for (int offset = 0; offset < listSize; offset++) {
            uidList.add(firstSeqUid + offset);
        }

        return uidList;
    }
```

用于生成指定秒`currentSecond`内的全部UID，提供给`BufferPaddingExecutor`进行填充。

## 总结

1. RIngBuffer的填充时机有3个：CachedUidGenerator时对RIngBuffer初始化、RIngBuffer#take()时检测达到阈值和周期性填充（如果有打开）。
2. RingBuffer的slots数组多读少写，不考虑伪共享问题。
3. JDK8中`-XX:-RestrictContended`搭配`@sun.misc.Contended`。