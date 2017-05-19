---
layout:     post
title:      "图解JDK并发工具源码 - AbstractQueuedSynchronizer Sync Queue"
subtitle:   "向Doug Lea学习并发工具设计"
author:     "Chris"
header-img: "img/post-bg-3.jpg"
tags:
    - Java
    - 源码研读
---

## 简介

根据[JSR-166](http://g.oswego.edu/dl/concurrency-interest/)规范，Java的1.5版本引入了JUC并发工具包，这个包主要维护着以下几个功能：内部同步状态的管理，同步状态的更新和检查操作，且至少有一个使得线程在同步状态被争夺时从而进入阻塞状态的方法，以及在其他线程改变这个同步状态时解除线程的阻塞。而AbstractQueuedSynchronizer（AQS）就是构建这些组件的基础框架，整个框架的关键就是如何管理被阻塞的线程的FIFO队列。

AbstractQueuedSynchronizer中的队列有两种

- 同步队列（Sync Queue） —  用于管理同步状态的**双向**链表
- 条件队列（Condition Queue）—  用于管理条件唤醒的**单向**链表

本文将AbstractQueuedSynchronizer简称为AQS，而同步队列和条件队列将使用它们的英语原意，Sync Queue和Condition Queue。

以下我们就开始通过源码分析层层深入地学习AQS中最重要的数据结构之一 — 同步队列 Sync Queue。

## AQS状态转换函数

- shouldParkAfterFailedAcquire(): <=0 —> signal
- cancelAcquire(): <=0->signal, all —> cancelled
- unparkSuccessor(): \<0 —> 0
- doReleaseShared(): signal —> 0, 0 —> propagate
- transferForSignal(): <=0 —> signal, condition —> 0
- fullyRelease(): all —> cancelled
- transferAfterCancelledWait(): condition —> 0
- addConditionWaiter(): 初始化Node为condition
- addWaiter(): 初始化Node为0

## 排他锁的获取

排他锁又称为独占锁，在AQS中的排他锁有3种获取方式，分别是：**忽略中断**，**响应中断**和**超时并响应中断**。由浅入深，下面的代码是**忽略中断**地获取排他锁的入口。

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
      // 使指定的节点自旋，并删除cancelled类型节点，将指定节点的前驱的waitStatus置为signal
      acquireQueued(
        // 添加当前线程包装为一个exclusive式的节点，添加至sync queue
        addWaiter(Node.EXCLUSIVE), arg))
    // Don't swallow interrupts
    selfInterrupt();
}
```

程序会先尝试通过tryAcquire获取锁，如果失败就进行addWaiter将当前线程添加至Sync Queue，并传入acquireQueued进行前驱节点的状态设置，并在合适的时机进行自我阻塞，等待唤醒，避免浪费CPU时间片。

需要说明有：

- arg：该参数与AQS的关键变量volatile int state有关，其含义根据不同的同步器而有所不同。
- tryAcquire(arg)：独占式获取同步状态，**由AQS子类实现**。

根据栈帧的调用顺序分析可知：

tryAcquire(int) -> **addWaiter(Node)** -> acquireQueued(Node, int)

### addWaiter(Node) - 将节点添加至Sync Queue

```java
private Node addWaiter(Node mode) {
  // 根据'当前线程'和'节点类型'构建节点
  Node node = new Node(Thread.currentThread(), mode);
  // Try the fast path of enq; backup to full enq on failure ->>
  // 尝试快速入队，如果失败则降级至full enq
  Node pred = tail;
  if (pred != null) {
    // 这里有点意思，先确定新的tail的prev引用指向旧的tail, 所以不要以为next域为null就是尾节点了
    node.prev = pred;
    // 防止有其他线程修改tail，使用CAS进行修改，如果失败则降级至full enq
    if (compareAndSetTail(pred, node)) {
      // 如果成功之后旧的tail的next指针再指向新的tail，成为双向链表
      pred.next = node;
      // 返回的是新的tail
      return node;
    }
  }
  // 如果队列为null或者CAS设置新的tail失败
  enq(node);
  return node;
}
```

整个方法的含义就是先尝试一次快速添加，如果出现数据竞争，那么才降级为自旋入队，enq(Node)其实才是真正的主角。当前的调用栈如下：

tryAcquire(int) -> addWaiter(Node) -> **enq(Node)** -> acquireQueued(Node, int)

### enq(Node) - 自旋入队

```java
private Node enq(final Node node) {
  for (; ; ) {
    // 重新获取tail节点
    Node t = tail;
    // 如果tail为null则说明队列首次使用，需要进行初始化
    if (t == null) { // Must initialize
      // 设置头节点，如果失败则存在竞争，留至下一轮循环
      if (compareAndSetHead(new Node()))
        // 初始化的时候head与tail具有相同的内存地址
        // b1
        tail = head;
    } else {
      // 这里与addWaiter(Node)方法下的部分代码一致
      node.prev = t;
      // b2 还未执行下面的CAS
      if (compareAndSetTail(t, node)) {
        // 本操作不能放与if块外，因为会引起tail节点的错误变更
        t.next = node;
        // 至此就是程序出口，至少有一个head和一个tail，而且两者的内存地址不一样，t就是node的前驱节点
        // b3
        return t;
      }
    }
  }
}
```

该方法可以直观地观察到Sync Queue的结构。在b1处Sync Queue处于初始化状态，将会继续循环；b2是中间态，会可能因为下面的CAS执行失败从而继续自旋；b3处才是真正的函数出口。

也就是说，当enq返回之后Sync Queue至少处于以下状态：

![](/img/in-post/aqs/addWaiter.png)

当前调用栈如下：

tryAcquire(int) -> addWaiter(Node) -> enq(Node) -> **acquireQueued(Node, int)**

### acquireQueued(Node, int) - 抢夺锁

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (; ; ) {
      // 获取prev节点，若为null即刻抛出NPE
      final Node p = node.predecessor();
      // 如果前驱为head才有资格进行锁的抢夺
      if (p == head && tryAcquire(arg)) {
        // 获取锁成功后就不需要再进行同步操作了，获取锁成功的线程作为新的head节点 ->>
        // 凡是head节点，head.thread与head.prev永远为null, 但是head.next不为null，waitStatus可能为0、propagate和signal的一种
        // release不负责出队，仅负责唤醒后继节点，setHead负责将head出队
        setHead(node);
        p.next = null; // help GC
        // 抢夺成功
        failed = false;
        // c1
        return interrupted;
      }
      // 如果前驱不是head，或者抢夺失败
      // 注意这里是if，而不是else if
      // c2
      if (// 根据节点的waitStatus决定是否需要挂起线程
        shouldParkAfterFailedAcquire(p, node) &&
        // 若前面为true，则执行挂起，待下次唤醒的时候检测中断的标志
        parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      // 如果抛出异常则取消锁的获取，进行出队(sync queue)操作
      cancelAcquire(node);
  }
}
```

当直接前驱节点为head的时候才具备锁的抢夺资格，当抢夺成功之后则将该节点置为新的head，虽然acquireQueued(Node, int)为忽略中断的方法，但友好地遵守了Don‘t swallow Interrupts原则，向上层返回中断标志。

下图是Sync Queue中的节点成功抢夺锁的队列状态（c1）

![](/img/in-post/aqs/acquireQueued-c1.png)

但并非前驱是head就一定会抢夺成功，正如此时外部有一个方法直接调用tryAcquire(arg)并且直接抢夺成功，那么原有在Sync Queue中具备抢夺资格的节点就要继续挂起（park），等待抢夺锁成功的线程调用release(int)来唤醒后继节点，这也是非公平锁的具体实现。

下图是前驱节点为head但是却抢夺锁失败的队列状态（c2）

![](/img/in-post/aqs/acquireQueued-c2.png)

至于那些前驱节点不是head的节点都不具备竞争资格，这些节点都将被挂起，由前驱节点负责唤醒。Sync Queue中的节点具有多种状态（waitStatus），如0、signal、propagate和cancelled，有如此多的节点状态，需要通过一个完整的判定来决定当前节点是否能被安全地挂起，执行这些判断的就是以下方法 — shouldParkAfterFailedAcquire

#### shouldParkAfterFailedAcquire(Node, Node) - 判断锁抢夺失败后是否该挂起

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    /*
     * This node has already set status asking a release
     * to signal it, so it can safely park.
     * 前驱节点的waitStatus为signal时可以放心地挂起当前node节点
     */
    // d1
    return true;
  if (ws > 0) {
    /*
     * Predecessor was cancelled. Skip over predecessors and
     * indicate retry.
     */
    do {
      node.prev = pred = pred.prev;
      // d2
    } while (pred.waitStatus > 0);
    // d3
    pred.next = node;
  } else {
    /*
     * waitStatus must be 0 or PROPAGATE.  Indicate that we
     * need a signal, but don't park yet.  Caller will need to
     * retry to make sure it cannot acquire before parking.
     */
    // 巧妙地复用了acquireQueue中的自旋，前驱的节点状态只能为0和propagate，其他状态都被上述逻辑所处理
    // d4
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

本方法借用了acquireQueued(Node, int)的自旋，确保删除cancelled节点，以及将前驱为0和propagate的节点置为signal，使用节点本身可以安全地挂起。其中有一点需要注意，在d2处的节点状态仅仅更新了prev域，其next域的更新会有延后。

下图为摘除cancelled时的可能状态（d2）

![](/img/in-post/aqs/shouldParkAfterFailedAcquire-d2.png)

在此刻，node.prev已经更新为有效的节点，但是pred.next仍未来得及更新，依旧指向旧的next节点

当shouldParkAfterFailedAcquire确认当前节点可以被安全挂起之后，由parkAndCheckInterrupt()执行具体的操作。

#### parkAndCheckInterrupt() - 挂起并检测中断

```java
private final boolean parkAndCheckInterrupt() {
  LockSupport.park(this);
  // p1
  return Thread.interrupted();
}
```

当前线程将会在此处被挂起，在被其他线程执行LockSupport.unpark(Thread)唤醒之后（p1），第一件事就是检查中断位，一方面以防下次调用LockSupport.park无效，其次可以将中断信息返回给调用者，由上层的调用者决定是否响应中断，而不是把中断标志掩埋。

### cancelAcquire(Node)

让我们回顾一下acquireQueued(Node, int)，分析一下可能抛出异常地方。

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    for (; ; ) {
      if (p == head && tryAcquire(arg)) {
		// a1 do sth
      }
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
        // a2 do sth
    }
  } finally {
    if (failed)
      // 如果抛出异常则取消锁的获取，也就是出队(sync queue)操作
      cancelAcquire(node);
  }

```

首次，可能由于实现者的粗心，在实现tryAcquire(int)的时候会意外抛出异常；其次，在a2处有可能会响应中断而抛出InterruptedException，如响应中断的获取锁方法doAcquireInterruptibly(int)；最后，如果是doAcquireNanos或doAcquireSharedNanos，在检测超时仍未获得锁，则会因为failed == true则在finally块中执行出队操作。不论在AQS中何种acquire类型的衍生方法，在进入自旋之前就会先将当前线程封装为Node节点，并进行入队操作。如果途中发生任何异常，则必须将该队列出队，实现这操作的方法cancelAcquire(Node)，下面我们来查看其源码。

```java
private void cancelAcquire(Node node) {
  // Ignore if node doesn't exist
  if (node == null)
    return;
  // node.thread只有在构造器，setHead和cancelAcquire这三个地方有设置
  // 被cancel的节点也很特殊，也会跟head一样将thread置为null
  node.thread = null;

  // Skip cancelled predecessors
  Node pred = node.prev;
  // 只有cancelled节点的waitStatus > 0, 这里跳过所有状态为cancelled的节点
  // 因为head绝不可能为cancelled，所以pred最低限度为head
  while (pred.waitStatus > 0)
    node.prev = pred = pred.prev;

  // predNext is the apparent node to unsplice. CASes below will
  // fail if not, in which case, we lost race vs another cancel
  // or signal, so no further action is necessary.
  Node predNext = pred.next;

  // Can use unconditional write instead of CAS here.
  // After this atomic step, other Nodes can skip past us.
  // Before, we are free of interference from other threads.
  // 将需要出队的节点的状态置为cancelled, 出队的操作会在shouldParkAfterFailedAcquire处顺便做了
  node.waitStatus = Node.CANCELLED;
  // If we are the tail, remove ourselves.
  // 如果要移除的节点就是tail，则移除自己 ->>
  // 如果compareAndSetTail失败，那就如上面的cancelled节点一样被跳过，可能这也是shouldParkAfterFailedAcquire从tail往前遍历的原因
  // e1 线程执行至此，或者是下面if语句的CAS失败
  if (node == tail && compareAndSetTail(node, pred)) {
    // e2 下面的cas失败也无妨，shouldParkAfterFailedAcquire会删除cancelled节点
    compareAndSetNext(pred, predNext, null);
    // e5: cas successfully
  } else {
    // If successor needs signal, try to set pred's next-link
    // so it will get one. Otherwise wake it up to propagate.
    int ws;
    // 这个if else结构的作用就是先找直接前驱看看是否能作为signal的节点，如果不能就只好唤醒当前最靠近被删除节点node的非cancel节点
    if (pred != head &&
        // 传入的node节点前的所有节点仅剩下initial, signal和propagate三种状态 ->>
        // 将前驱节点的waitStatus<=0的转换为signal
        ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
        // 只有head或者被cancel节点的thread为null，防止数据竞争下有前驱节点被置为cancelled的情况
        pred.thread != null) {
      Node next = node.next;
      // e3
      // 如果node为tail但是上面compareAndSetTail失败就会存在next==null的情况，如果直接后继节点为cancelled的话就保持e3+e6的状态
      if (next != null && next.waitStatus <= 0)
        // 其实这里跨度非常大，同时node.prev还没设置
        // e4: 下面的cas成功
        compareAndSetNext(pred, predNext, next);
    } else {
      // 如果是新的前驱是头结点，或者新的前驱不是头结点但他的waitStatus为propagate或0且设置为signal失败 ->>
      // 这里有可能唤醒sync queue中除head的节点
      unparkSuccessor(node);
    }
    // 执行到这一步, 被cancelled的节点无论是tail还是其他节点，都会将自身的next指向回自己，所以只有prev是准确的
    node.next = node; // help GC
    // e6
  }
}
```

方法相对冗长，但概括来说就是先删除前驱为cancelled状态的节点，然后尝试将清理后的队列中的**有效的**直接前驱设置其状态为signal，但如果传入的节点正是head（无法cancelled的节点，只能出队）或者是有线程竞争临界资源导致CAS，那么将会直接唤醒其后继的**有效的**节点，使其后继节点继续在自旋中通过shouldParkAfterFailedAcquire更改有效前驱的状态。以上这些操作都是为了避免后继节点一直挂起，无法被唤醒的状况。按照静态代码的推测，这段代码的运行有很多中间态，这里挑出其中几点进行说明。

下图展示的是被cancel的节点为tail，但是在e1下的cas失败，进而进入e3的状况（e1+e3+e6）

![](/img/in-post/aqs/cancelAcquire-e1-e3-e6.png)

如果你细心看，你一定会觉得很奇怪，为什么T2的next还是指向cancelled的节点而没有被更新？其实这个还需要与下面将要介绍的方法unparkSuccessor(Node)一起思考，如果尝试唤醒直接后继节点将会发现他的waitStatus为cancelled，此时AQS采取的策略是会从tail往前遍历寻找最合适的节点来进行唤醒，而不是依赖于next域，这样就解决了问题。

如果被取消的节点不为tail且节点状态不为cancelled，直接前驱不为head，而且设置前驱的waitStatus成功或者前驱的waitStatus正是signal的时候，有如下状态（e4+e6）

![](/img/in-post/aqs/cancelAcquire-e4-e6.png)

可以看出，如果从tail往前迭代，依然可以找到node节点，该节点会在shouldParkAfterFailedAcquire和cancelAcquire中被删除。并且通过源码分析我们可以知道，当一个节点从tail往前遍历都无法寻获的话，那么该节点就一定不存在于Sync Queue，而next域更像是一个辅助节点，并不能提供一个准确的线索。

## 排他锁的释放

```java
public final boolean release(int arg) {
  // tryRelease由子类实现, arg为释放的层数
  if (tryRelease(arg)) {
    // 同步队列仅为头结点拥有锁
    Node h = head;
    // head的waitStatus为0说明没有节点被添加至sync queue
    // head的waitStatus为0的话说明head没有后继节点，不需要执行唤醒
    if (h != null && h.waitStatus != 0)
      // 唤醒successor
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

tryRelease(int)由子类实现，当Sync Queue中仅有head的时候不需要唤醒后继节点，因为队列就只有这一个哑元。我们现在来看看具体的唤醒过程。

```java
private void unparkSuccessor(Node node) {
  /*
   * If status is negative (i.e., possibly needing signal) try
   * to clear in anticipation of signalling.  It is OK if this
   * fails or if status is changed by waiting thread.
   * 将当前节点的waitStatus置为0
   * 这里可能的值就是signal和propagate
   */
  int ws = node.waitStatus;
  // 通知完一次就重置为0，但并不保证成功
  if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);

  /*
   * Thread to unpark is held in successor, which is normally
   * just the next node.  But if cancelled or apparently null,
   * traverse backwards from tail to find the actual
   * non-cancelled successor.
   */
  Node s = node.next;
  // waitStatus中只有cancelled大于0
  // 看过cancelAcquire就知道为什么从tail往前遍历
  if (s == null || s.waitStatus > 0) {
    // 如果后继节点为null，或者他的状态为cancelled，则从tail往前迭代，找到最靠近node且waitStatus不为cancelled的节点, 时间复杂度O(n)
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
      // 这里的语义很明确，将sync queue(里面不存在condition节点)里的0，signal，propagate节点都可以进行唤醒
      // 0可能是从condition queue中通过enq入队，仍未来得及设置前驱节点waitStatus的情况
      if (t.waitStatus <= 0)
        // f1
        s = t;
  }
  if (s != null)
    // 如果找到符合的节点，则进行唤醒, s不一定是直接后继节点
    LockSupport.unpark(s.thread);
}
```

其前驱节点在唤醒后继线程之前，将waitStatus重置为0（并不保证一定会设置成功，但是失败也没有影响，因为该node一般会是head，即将会被出队），表示将完成signal的任务，但CAS是否成功程序并不关心，失败是被允许的。然后取出当前node的后继，从tail往前遍历，寻找一个最接近node的节点，然后唤醒对应线程。

## 共享锁的获取

排他锁的特点是，在当前线程竞争锁成功之后，直至成功释放之前，其他线程均处于挂起的状态。而共享锁则不一样，共享锁的本意是多个线程能在（几近）同一时刻获取到锁，所以在锁竞争成功之后会继续唤醒后继节点，可以看做是一种蔓延或者是扩散。共享锁的获取与排他锁大同小异，我们先来看看源码。

```java
private void doAcquireShared(int arg) {
  // 添加至sync queue的操作在这里
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (; ; ) {
      final Node p = node.predecessor();
      if (p == head) {
        // 如果前驱节点是头结点就进行锁的抢夺
        // 返回0和大于0也算成功，但是具体含义不同
        int r = tryAcquireShared(arg);
        if (r >= 0) {
          // 获取锁成功之后应该陆续唤醒其他节点
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          if (interrupted)
            selfInterrupt();
          failed = false;
          return;
        }
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

本方法虽然会忽略中断，但是会保留中断信息，可以看出除了setHeadAndPropagate(Node, int)方法其他大部分均与排他锁的获取一致。另外需要注意的是，添加至Sync Queue的节点类型是SHARED。

#### setHeadAndPropagate(Node, int) - 节点出队并传播状态

```java
private void setHeadAndPropagate(Node node, int propagate) {
  // h在这里是旧的head
  Node h = head; // Record old head for check below
  setHead(node);
  // 外面传入的propagate是>=0, >0是一回事，=0是另一回事 ->>
  // 注意这里检测的是旧的head的waitStatus
  if (propagate > 0 || h == null || h.waitStatus < 0) {
    // 新head的下一个节点
    Node s = node.next;
    // 如果下一个节点为null可能是enq还未执行完成
    if (s == null || s.isShared())
      // 在releaseShared和setHeadAndPropagate之后都会调用这个方法
      // 注意是node的下一个节点为shared模式才会唤醒, 这里的node.next通常就是doReleaseShared中要唤醒的对象
      // g1
      doReleaseShared();
  }
}
```

本方法先将旧的head出队，将当前获得锁的线程置为新的head，同时**检测旧的head的waitStatus是否为signal或propagate**。如果waitStatus的层数或释放的层数大于0，**才有可能唤醒下一个shared节点**。

<u>但我有一个疑问，在if条件语句中h==null是什么意思？</u>

需要注意的是该函数的形参int propagate为外面tryAcquireShared(int)所返回的值。我们细看一下其java doc

```java
* @return a negative value on failure; zero if acquisition in shared
* mode succeeded but no subsequent shared-mode acquire can
* succeed; and a positive value if acquisition in shared
* mode succeeded and subsequent shared-mode acquires might
* also succeed, in which case a subsequent waiting thread
* must check availability. (Support for three different
* return values enables this method to be used in contexts
* where acquires only sometimes act exclusively.)  Upon
* success, this object has been acquired.
```

负数为失败；0表示在共享锁模式下获取锁成功，但是不需要后继节点获取共享锁；正数为共享锁获取成功并且需要传播。

**<u>鄙人认为</u>**在g1处会有一个临界状态，若node的直接后继节点s是一个响应中断和超时失败的节点（即通过doAcquireNanos或doAcquireSharedNanos生成），当s的线程被中断后抛出异常，此时s将会进入cancelAcquire并修正自己的waitStatus为cancelled（根据happens-before规则，修正修cancelled的执行语句应该在更新其prev引用之后，但不排除处理器重排序的可能），而这个时候在doReleaseShared()中唤醒的就**可能就是exclusive节点**，而造成意料之外的状况。

因此对于一个Sync Queue中同时存在shared和exclusive节点的时候需要特别注意：

- 如果Sync Queue中同时存在shared和exclusive，应该考虑是否需要禁用响应中断的接口。
- 或许shared和exclusive根本就不应该存在于同一Sync Queue，通过tryAcquire或tryAcquireShared及其一系列衍生方法控制节点的入队。

### doReleaseShared()

```java
private void doReleaseShared() {
  /*
   * Ensure that a release propagates, even if there are other
   * in-progress acquires/releases.  This proceeds in the usual
   * way of trying to unparkSuccessor of head if it needs
   * signal. But if it does not, status is set to PROPAGATE to
   * ensure that upon release, propagation continues.
   * Additionally, we must loop in case a new node is added
   * while we are doing this. Also, unlike other uses of
   * unparkSuccessor, we need to know if CAS to reset status
   * fails, if so rechecking.
   * 整个函数的作用就是将head的waitStatus为signal的置为0，然后唤醒后面的有效节点。如果head的waitStatus为0则修改为propagate。
   */
  // 这里是更新head自己的状态
  for (; ; ) {
    Node h = head;
    // 因为是要将后续节点释放，所如果head为null或者是初始化的状态(head==tail)的话则没有必要再进行判断
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      if (ws == Node.SIGNAL) {
		// h0
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          continue;            // loop to recheck cases
        // h1
        unparkSuccessor(h);
      } else if (ws == 0 &&
                 // 0转换成propagate，让setHeadAndPropagate可以继续调用doReleaseShared, 这个可能就是所谓的传递->>
                 // 一般的头节点就是signal和propagate(后面的节点还来不及enq)，在doReleaseShared中旧的head出队之后检测出为propagate，就会调用doReleaseShare，让新的head的后继节点唤醒
                 // head为0的后继节点必然不会park，所以可以安心设置。如果被其他线程更改waitStatus会继续自旋
                 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        // h2
        continue;                // loop on failed CAS
    }
    // 如果head被出队，那么继续获取新的head再进行判断
    if (h == head)                   // loop if head changed
      break;
  }
}
```

这是一个存在数据竞争的方法，可能由共享模式下的链式唤醒导致多个线程竞争，所以head和head.waitStatus都有可能变更，需要通过CAS自旋来解决。

head只能为0、signal和propagate这3种状态，当head为propagate的状态一般是：有节点还未入队，所以此时Sync Queue只有一个dummy header node。但是为了将共享锁的释放状态传递下去，**必须**将head置为propagate，如此以往，之前还未来得及入队的节点在执行完addWaiter(Node.SHARED)和setHeadAndPropagate(Node, int)之后判断出`h.waitStatus < 0`继而继续调用doReleaseShared()。当然，这一系列操作不包含额外的线程直接获取锁的操作，因为其调用acquireShared(int)中的if块中的tryAcquireShared(int)就直接成功了，根本没必要进入Sync Queue排队。

## Sync Queue特点总结

- head为哑元
- head.prev == null && head.thread == null
- tail.next = null
- Sync Queue的合法节点状态0, signal, propagate, cancelled
- 在shouldParkAfterFailedAcquire和cancelAcquire两个函数中会顺便整理cancelled节点
- unparkSuccessor被doReleaseShared, cancelAcquire和release三个方法调用
- 共享锁propagate有两条出路: shouldParkAfterFailedAcquire, transferForSignal。而后者是Condition Queue转换至Sync Queue所用，所以唯一的出路就是前者。
- waitStatus为0一般为tail或者为倒数第二个节点（还未来得及被tail设置前驱状态的节点），head可以为0、signal和propagate（共享锁模式下）。
- Node.next == null的情况
  - node == tail
  - enq时还未来得及设置next域
  - 在setHead(Node)后就的head的next被置为null，p.next = null

## 后话

至此，关于Sync Queue的源码介绍已经完毕，下一篇将会介绍AQS中的另一个组件Condition Queue。