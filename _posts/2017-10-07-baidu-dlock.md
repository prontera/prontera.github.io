---
layout:     post
title:      "百度 - DLock源码解析"
subtitle:   "Understand how Baidu DLock works"
author:     "Chris"
header-img: "img/post-bg-1.jpg"
tags:
    - Java
    - 源码研读
---

## 简介

[DLock](https://github.com/baidu/dlock)是由Java实现的，一套高效高可靠的分布式锁方案。 使用Redis存储锁，通过[Lua]([https://en.wikipedia.org/wiki/Lua_(programming_language))脚本进行原子性锁操作，实现了基于Redis过期机制的[lease]([https://en.wikipedia.org/wiki/Lease_(computer_science))，并提供了一种基于变种CLH队列的进程级锁竞争模型。

整个组件的架构如下图所示

![](https://github.com/baidu/dlock/blob/master/doc/dlock-architecture.png?raw=true)

由于Github上的描述过于简略，并为了保持对文章的严谨性，在完整地读过源码后，我先简单地描述一下各个角色在该组件中的位置与功能。

该组件中的类CLH队列实质是AQS Sync Queue简略版，但因为锁是非完全进程内可控，所以会导致无法唤醒的窘境，为解决这个问题所以引入了`Retry Thread`，该线程会以poll的方式轮询Redis锁是否可用；在分布式的环境环境下无可避免地要面对网络分区和网络抖动的问题，对于分布式锁来说就是**续约**问题，dlock会以**提前续约**的方式来**尽量**确保续约成功，该职责由`Lease Thread`完成；至于在抢夺锁成功之后，进程内由`Exclusive Thread`确认获得锁的线程。

在大致了解图上的角色之后，接下来我们开始分析源码。

## 源码解析

本文基于commit id `9b8a82a0da327c3a4dc7128ad0707d26802f3b43`所写，为编写本文时（2017-10-07 15:51:25）的最新Master分支，阅读时须注意未来的版本迭代有可能造成功能上的差异。

### 目录结构

```shell
.
├── DistributedReentrantLock.java		- AQS的同步队列实现
├── domain
│   ├── DLockConfig.java			- 锁的配置，如key和lease time
│   ├── DLockEntity.java			- 锁的实体，主要是锁的value(locker)
│   ├── DLockStatus.java			- 锁的状态
│   └── DLockType.java				- 锁的类型
├── exception
│   ├── DLockProcessException.java		- RedisProcessException父类
│   ├── OptimisticLockingException.java		- redis命令正确但输入非我们所期望
│   └── RedisProcessException.java		- jedis执行错误
├── jedis
│   └── JedisClient.java
├── processor
│   ├── DLockProcessor.java			- 锁操作接口，如读取、设置、续约
│   └── impl
│       └── RedisLockProcessor.java		- 锁操作实现类
├── support
│   └── DLockGenerator.java			- 组合DLockConfig和DistributedReentrantLock的工具
└── utils
    ├── EnumUtils.java
    ├── NetUtils.java
    ├── ReflectionUtils.java
    └── ValuedEnum.java
```

### 组件功能描述

#### DLockConfig - key与续约时间的Holder

```java
public class DLockConfig implements Serializable {
    private static final long serialVersionUID = -1332663877601479136L;

    /**
     * Prefix for unique key generating
     */
    public static final String UK_PRE = "DLOCK";

    /**
     * Separator for unique key generating
     */
    public static final String UK_SP = "_";

    /**
     * Lock type represents a group lockTargets with the same type.
     * The type is divided by different business scenarios, kind of USER_LOCK, ORDER_LOCK, BATCH_PROCCESS_LOCK...
     */
    private final String lockType;

    /**
     * Lock target represents a real lock target. lockType: USER_LOCK, lockTarget should be the UserID.
     *
     * @see DLockEntity#locker
     */
    private final String lockTarget;

    /**
     * Lock unique key represents the minimum granularity of the lock.
     * The naming policy is $UK_PRE_$lockType_$lockTarget
     */
    private final String lockUniqueKey;

    /**
     * Lock lease duration
     */
    private final int lease;

    /**
     * Lock Lease time unit
     */
    private final TimeUnit leaseTimeUnit;

    /**
     * Constructor with lockType & lockTarget & leaseTime & leaseTimeUnit
     */
    public DLockConfig(String lockType, String lockTarget, int lease, TimeUnit leaseTimeUnit) {
        this.lockType = lockType;
        this.lockTarget = lockTarget;
        this.lockUniqueKey = UK_PRE + UK_SP + lockType + UK_SP + StringUtils.trimToEmpty(lockTarget);
        this.lease = lease;
        this.leaseTimeUnit = leaseTimeUnit;
    }
	
  	// getter and setter
}
```

该类存储key的构造方式，最终输出为`lockUniqueKey`，也就是说redis key为`"DLOCK_" + lockType + "_" + StringUtils.trimToEmpty(lockTarget)`，而redis value则存储在`DLockEntity`。

#### DLockEntity - 锁的元数据

```java
public class DLockEntity implements Serializable, Cloneable {
    private static final long serialVersionUID = 8479390959137749786L;

    /**
     * Task status default as {@link DLockStatus#INITIAL}
     */
    private DLockStatus lockStatus = DLockStatus.INITIAL;

    /**
     * The server ip address that locked the task
     */
    private String locker;

    /**
     * Lock time for milliseconds
     */
    private Long lockTime = -1L;
	
  	// getter and setter
}
```

该类的最主要作用是存储其分布式锁的redis value，也就是属性`locker`。

在目前版本来看`lockStatus`和`lockTime`并无用处（是的，我就是这么肯定）。从DLock的整体设计上来说，其开始的目标是支持DB和Redis实现，而这两个属性正是预留所用。

#### RedisLockProcessor - redis锁命令执行者

##### 读取锁

该读取操作完成后仅直接封装locker，其他属性为人工填充。

```java
    /**
     * Load by unique key. For redis implement, you can find locker & status from the result entity.
     *
     * @param uniqueKey key
     * @throws RedisProcessException if catch any exception from {@link redis.clients.jedis.Jedis}
     */
    @Override
    public DLockEntity load(String uniqueKey) throws RedisProcessException {
        // GET command
        String locker;
        try {
            locker = jedisClient.get(uniqueKey);
        } catch (Exception e) {
            LOGGER.warn("Exception occurred by GET command for key: {}", uniqueKey, e);
            throw new RedisProcessException("Exception occurred by GET command for key:" + uniqueKey, e);
        }

        if (locker == null) {
            return null;
        }

        // build entity
        DLockEntity lockEntity = new DLockEntity();
        lockEntity.setLocker(locker);
        lockEntity.setLockStatus(DLockStatus.PROCESSING);

        return lockEntity;
    }
```

##### 设置锁

调用setnx命令进行锁的抢夺，pexpire设置以毫秒为单位的过期时间。

```java
    /**
     * Update for lock using redis SET(NX, PX) command.
     *
     * @param newLock    with locker in it
     * @param lockConfig
     * @throws RedisProcessException      Redis command execute exception
     * @throws OptimisticLockingException the lock is hold by the other request.
     */
    @Override
    public void updateForLock(DLockEntity newLock, DLockConfig lockConfig)
            throws RedisProcessException, OptimisticLockingException {
        // SET(NX, PX) command
        String lockRes;
        try {
            lockRes = jedisClient.set(lockConfig.getLockUniqueKey(), newLock.getLocker(), SET_ARG_NOT_EXIST,
                    SET_ARG_EXPIRE, lockConfig.getMillisLease());

        } catch (Exception e) {
            LOGGER.warn("Exception occurred by SET(NX, PX) command for key: {}", lockConfig.getLockUniqueKey(), e);
            throw new RedisProcessException(
                    "Exception occurred by SET(NX, PX) command for key:" + lockConfig.getLockUniqueKey(), e);
        }

        if (!RES_OK.equals(lockRes)) {
            LOGGER.warn("Fail to get lock for key:{} ,locker={}", lockConfig.getLockUniqueKey(), newLock.getLocker());
            throw new OptimisticLockingException(
                    "Fail to get lock for key:" + lockConfig.getLockUniqueKey() + " ,locker=" + newLock.getLocker());
        }
    }
```

##### 锁的过期/续约

这里使用了lua以达到原子性事务的作用。

```java
    /**
     * Extend lease for lock with lua script.
     *
     * @param leaseLock  with locker in it
     * @param lockConfig
     * @throws RedisProcessException      if catch any exception from {@link redis.clients.jedis.Jedis}
     * @throws OptimisticLockingException if the lock is released or be hold by another one.
     */
    @Override
    public void expandLockExpire(DLockEntity leaseLock, DLockConfig lockConfig)
            throws RedisProcessException, OptimisticLockingException {
        // Expire if key is existed and equal with the specified value(locker).
        // pexpire的控制单位为毫秒，expire的控制单位为秒
        String leaseScript = "if (redis.call('get', KEYS[1]) == ARGV[1]) then "
                + "    return redis.call('pexpire', KEYS[1], ARGV[2]); "
                + "else"
                + "    return nil; "
                + "end; ";

        Object leaseRes;
        try {
            leaseRes = jedisClient.eval(leaseScript, Arrays.asList(lockConfig.getLockUniqueKey()),
                    Arrays.asList(leaseLock.getLocker(), lockConfig.getMillisLease() + ""));
        } catch (Exception e) {
            LOGGER.warn("Exception occurred by ExpandLease lua script for key:" + lockConfig.getLockUniqueKey(), e);
            throw new RedisProcessException(
                    "Exception occurred by ExpandLease lua script for key:" + lockConfig.getLockUniqueKey(), e);
        }

        // null means lua return nil (the lock is released or be hold by the other request)
        if (leaseRes == null) {
            LOGGER.warn("Fail to lease for key:{} ,locker={}", lockConfig.getLockUniqueKey(), leaseLock.getLocker());
            throw new OptimisticLockingException(
                    "Fail to lease for key:" + lockConfig.getLockUniqueKey() + " ,locker=" + leaseLock.getLocker());
        }
    }
```

##### 锁的释放

这里同样使用了lua以达到原子性事务的作用。

```java
    /**
     * Release lock using lua script.
     *
     * @param currentLock with locker in it
     * @param lockConfig
     * @throws RedisProcessException      if catch any exception from {@link redis.clients.jedis.Jedis}
     * @throws OptimisticLockingException if the lock is released or be hold by another one.
     */
    @Override
    public void updateForUnlock(DLockEntity currentLock, DLockConfig lockConfig)
            throws RedisProcessException, OptimisticLockingException {
        // Delete if key is existed and equal with the specified value(locker).
        String unlockScript = "if (redis.call('get', KEYS[1]) == ARGV[1]) then "
                + "    return redis.call('del', KEYS[1]); "
                + "else "
                + "    return nil; "
                + "end;";

        Object unlockRes;
        try {
            unlockRes = jedisClient.eval(unlockScript, Arrays.asList(lockConfig.getLockUniqueKey()),
                    Arrays.asList(currentLock.getLocker()));
        } catch (Exception e) {
            LOGGER.warn("Exception occurred by Unlock lua script for key:" + lockConfig.getLockUniqueKey(), e);
            throw new RedisProcessException(
                    "Exception occurred by Unlock lua script for key:" + lockConfig.getLockUniqueKey(), e);
        }

        // null means lua return nil (the lock is released or be hold by the other request)
        if (unlockRes == null) {
            LOGGER.warn("Fail to unlock for key:{} ,locker={}", lockConfig.getLockUniqueKey(), currentLock.getLocker());
            throw new OptimisticLockingException("Fail to unlock for key:" + lockConfig.getLockUniqueKey()
                    + ",locker=" + currentLock.getLocker());
        }
    }
```

#### DistributedReentrantLock - 可重入的分布式锁

此类为DLock组件的核心部分，与AQS一样使用了无锁的抢夺方式。

其中字段的含义如下

```java
    /**
     * Lock configuration(存储redis key和lease time)
     */
    private final DLockConfig lockConfig;
    /**
     * Lock processor(redis lock service/processor)
     */
    private final DLockProcessor lockProcessor;

    /**
     * Head of the wait queue, lazily initialized. Except for initialization, it is modified only via method setHead.
     * Note: If head exists, its waitStatus is guaranteed not to be CANCELLED.(sync queue队列头结点)
     */
    private final AtomicReference<Node> head = new AtomicReference<>();
    /**
     * Tail of the wait queue, lazily initialized. Modified only via method enq to add new wait node.(sync queue队列尾节点)
     */
    private final AtomicReference<Node> tail = new AtomicReference<>();

    /**
     * The current owner of exclusive mode synchronization.(获得锁的线程)
     */
    private final AtomicReference<Thread> exclusiveOwnerThread = new AtomicReference<>();
    /**
     * Retry thread reference(进程间锁状态检测所用)
     */
    private final AtomicReference<RetryLockThread> retryLockRef = new AtomicReference<>();
    /**
     * Expand lease thread reference(自动续约线程)
     */
    private final AtomicReference<ExpandLockLeaseThread> expandLockRef = new AtomicReference<>();

    /**
     * Once a thread hold this lock, the thread can reentrant the lock.
     * This value represents the count of holding this lock. Default as 0(可重入锁计数器)
     */
    private final AtomicInteger holdCount = new AtomicInteger(0);

```

##### AQS Sync Queue入队操作

因为分布式锁的跨进程特性，在当前进程中没有获取到锁后则启动RetryThread，以防当前**进程**中Sync Queue元素无法被唤醒，然后再挂起当前**线程**。

```java
    final void acquireQueued(final Node node) {
        for (; ; ) {
            final Node p = node.prev.get();
            if (p == head.get() && tryLock()) {
                head.set(node);
                p.next.set(null); // help GC
                break;
            }

            // if need, start retry thread
            // 获取到锁的时候会设置为当前线程，在释放锁的时候会设置为null
            if (exclusiveOwnerThread.get() == null) {
                // 进程级别的锁，因为不能跨进程通知，所以用poll的方式唤醒
                startRetryThread();
            }

            // park current thread
            LockSupport.park(this);
        }
    }
```

`retry thread`首次将会在十分之一的`lease time`时间启动，以后每隔六分之一个`lease time`单位时间进行轮询。

```java
retryLockRef.compareAndSet(t, new RetryLockThread((int) (lockConfig.getMillisLease() / 10), (int) (lockConfig.getMillisLease() / 6)));
```

在检测到分布式锁被释放之后马上唤醒Sync Queue中的头结点以进行锁的抢夺。

##### 抢夺锁

```java
    /**
     * Lock redis record through the atomic command Set(key, value, NX, PX, expireTime), only one request will success
     * while multiple concurrently requesting.
     */
    @Override
    public boolean tryLock() {

        // current thread can reentrant, and locked times add once
      	// 可重入特性
        if (Thread.currentThread() == this.exclusiveOwnerThread.get()) {
            this.holdCount.incrementAndGet();
            return true;
        }

        DLockEntity newLock = new DLockEntity();
        newLock.setLockTime(System.currentTimeMillis());
        // IP_ThreadId
        newLock.setLocker(generateLocker());
        newLock.setLockStatus(DLockStatus.PROCESSING);

        boolean locked = false;
        try {
            // get lock directly
            lockProcessor.updateForLock(newLock, lockConfig);
            locked = true;

        } catch (OptimisticLockingException | DLockProcessException e) {
            // NOPE. Retry in the next round.
        }

        if (locked) {
            // set exclusive thread
            this.exclusiveOwnerThread.set(Thread.currentThread());

            // locked times reset to one(可重入计数器)
            this.holdCount.set(1);

            // shutdown retry thread(已经成功获取锁则不用再次重试了)
            shutdownRetryThread();

            // start the timer for expand lease time(开启自动续约线程)
            startExpandLockLeaseThread(newLock);
        }

        return locked;
    }
```

结合上述的组件架构图来看，`exclusive owner thread`将会在成功抢夺锁后被设置，此时就不再需要`retry thread`去监听锁的状态，并且启动`lease thread`以进行续约。

`lease thread`将会即时启动，以后每隔75%的lease time时间进行续约。

```java
// set new expand lock thread
int retryInterval = (int) (lockConfig.getMillisLease() * 0.75);
expandLockRef.compareAndSet(t, new ExpandLockLeaseThread(lock, 1, retryInterval));
```

##### 释放锁

```java
    /**
     * Attempts to release this lock.<p>
     * <p>
     * If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *                                      hold this lock
     */
    @Override
    public void unlock() throws IllegalMonitorStateException {
        // lock must be hold by current thread
        if (Thread.currentThread() != this.exclusiveOwnerThread.get()) {
            throw new IllegalMonitorStateException();
        }

        // lock is still be hold
        if (holdCount.decrementAndGet() > 0) {
            return;
        }

        // clear remote lock
        DLockEntity currentLock = new DLockEntity();
        currentLock.setLocker(generateLocker());
        currentLock.setLockStatus(DLockStatus.PROCESSING);

        try {
            // release remote lock(删除当前锁)
            lockProcessor.updateForUnlock(currentLock, lockConfig);

        } catch (OptimisticLockingException | DLockProcessException e) {
            // NOPE. Lock will deleted automatic after the expire time.

        } finally {
            // Release exclusive owner
            this.exclusiveOwnerThread.compareAndSet(Thread.currentThread(), null);

            // Shutdown expand thread
            shutdownExpandThread();

            // wake up the head node for compete lock
            unparkQueuedNode();
        }
    }
```

在释放锁以后会调用`shutdownExpandThread()`中断续约线程

```java
    /**
     * Shutdown retry thread
     */
    private void shutdownRetryThread() {
        RetryLockThread t = retryLockRef.get();
        if (t != null && t.isAlive()) {
            t.interrupt();
        }
    }
```

`lease thread`的睡眠使用`wait()`实现，所以当其被中断的时候会将会抛出`InterruptedException`，此时通过变量控制可优雅地中止该任务。

```java
        @Override
        public void run() {
            while (!shouldShutdown) {
                synchronized (sync) {
                    try {
                        // first running, delay
                        if (firstRunning && delay > 0) {
                            firstRunning = false;
                            sync.wait(delay);
                        }

                        // execute task
                        execute();

                        // wait for interval
                        sync.wait(retryInterval);

                    } catch (InterruptedException e) {
                        shouldShutdown = true;
                    }
                }
            }

            // clear associated resources for implementations
            beforeShutdown();
        }
```

在中止`lease thread`之后唤醒Sync Queue的头结点，重新回到了锁的竞争之上，至此DLock的核心实现就全部解析完成。