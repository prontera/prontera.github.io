---
layout:     post
title:      "字解JDK并发工具源码 - AbstractQueuedSynchronizer Condition Queue"
subtitle:   "向Doug Lea学习并发工具设计"
author:     "Chris"
header-img: "img/post-bg-2.jpg"
tags:
    - Java
    - 源码研读
---

![](/img/in-post/aqs/condition-queue.png)

## 简介

Condition是一个接口，其子类在AbstractQueuedSynchronizer中实现，名为ConditionObject。

在本文中如没有特殊说明的话Condition皆为ConditionObject，他的作用与Object提供的监视器方法一样，实现等待通知模式，在Object中wait()、notify()和notifyAll()分别对应Condition中的await()、signal()和signalAll()。但是Condition却可以提供忽略中断的等待方法，并且可以创建多个等待条件。

## await() - 睡眠

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();

  Node node = addConditionWaiter();
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    // k1
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  if (//k2
    acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

我们需要的是各个击破，当然更需要一些预备的知识让我们更好地理解await()，当前调用顺序如下：

**addConditionWaiter()** -> fullyRelease(Node) -> isOnSyncQueue(Node) -> checkInterruptWhileWaiting(Node)

### addConditionWaiter() - 添加元素至Condition Queue

```java
private Node addConditionWaiter() {
  Node t = lastWaiter;
  // If lastWaiter is cancelled, clean out.
  // 根据上面的英文描述，推断condition queue中的节点类型只有两种: condition, cancelled
  if (t != null && t.waitStatus != Node.CONDITION) {
    unlinkCancelledWaiters();
    // 清除cancelled后重新获取WaitQueue中的尾节点
    t = lastWaiter;
  }
  // 构建WaitQueue的预节点。Node.nextWater的属性在sync queue表示节点类型mode，即exclusive还是shared；在condition queue中都是exclusive，nextWaiter用作本意--下一个waiter。
  Node node = new Node(Thread.currentThread(), Node.CONDITION);
  // 如果尾节点引用为null则说明队列未被初始化
  if (t == null)
    firstWaiter = node;
  else
    t.nextWaiter = node;
  lastWaiter = node;
  // i1
  return node;
}
```

每一次添加节点到Condition Queue之前都会先进行一次整理，删除lastWaiter前面的cancelled类型节点。然后再其链接到Condition Queue的尾端，完成真正的入队操作。

#### unlinkCancelledWaiters() - 从前往后删除cancelled节点

```java
private void unlinkCancelledWaiters() {
  Node t = firstWaiter;
  Node trail = null;
  while (t != null) {
    Node next = t.nextWaiter;
    if (t.waitStatus != Node.CONDITION) {
      // 不是condition就是cancelled，所以这里直接把t.nextWaiter置为null
      t.nextWaiter = null;
      if (trail == null)
        firstWaiter = next;
      else
        trail.nextWaiter = next;
      if (next == null)
        lastWaiter = trail;
    } else
      trail = t;
    t = next;
  }
}
```

Condition Queue是一个单向链表所以清理工作是从前往后的，而且await()是在获得排他锁之后方可调用，所以这些操作也是线程安全的。这个方法除了在addConditionWaiter()中调用以外，还会在await()唤醒后再调用一次。

至此完成了addConditionWaiter()，将视野拉到await()，当前的调用顺序如下：

addConditionWaiter() -> **fullyRelease(Node)** -> isOnSyncQueue(Node) -> checkInterruptWhileWaiting(Node)

## fullyRelease(Node) - 释放锁从Sync Queue中出队

```java
final int fullyRelease(Node node) {
  boolean failed = true;
  try {
    int savedState = getState();
    // 释放在Sync Queue的节点
    if (release(savedState)) {
      failed = false;
      return savedState;
    } else {
      throw new IllegalMonitorStateException();
    }
  } finally {
    if (failed)
      node.waitStatus = Node.CANCELLED;
  }
}
```

回顾一下await()的调用顺序，我们先将当前线程封装成Node添加至Condition Queue，在未执行fullyRelease(Node)之前，该节点同时存在于Condition Queue与Sync Queue（尽管head.thread为null，但事实上就是同一线程），所以此时要将Sync Queue中的head出队，并保存其“释放的层数”以便于重新入队，如果在release期间有任何异常或者是release返回false则直接将Condition Queue中的节点置为cancelled，由unlinkCancelledWaiters()进行清理。

当前的调用顺序如下：

addConditionWaiter() -> fullyRelease(Node) -> **isOnSyncQueue(Node)** -> checkInterruptWhileWaiting(Node)

## isOnSyncQueue(Node) - 判断节点是否处于Sync Queue

```java
final boolean isOnSyncQueue(Node node) {
  // 在sync queue中的节点有若干特点，waitStatus无condition，node.prev一定不为null
  if (node.waitStatus == Node.CONDITION || node.prev == null)
    return false;
  // 如果next不为null则一定在sync queue
  if (node.next != null) // If has successor, it must be on queue
    return true;
  /*
   * node.prev can be non-null, but not yet on queue because
   * the CAS to place it on queue can fail. So we have to
   * traverse from tail to make sure it actually made it.  It
   * will always be near the tail in calls to this method, and
   * unless the CAS failed (which is unlikely), it will be
   * there, so we hardly ever traverse much.
   * waitStatus不为condition，node.prev不为null，但是node.next为null，根据enq方法，应该是未完成入队操作，所以这里降级为全sync queue搜索
   */
  return findNodeFromTail(node);
}
```

本方法保证所传入的形参node一定不为head，要判断是否在Sync Queue那就要分析他和Condition Queue的特点了。Condition Queue中特有condition节点，而且prev域一定为null；如果next不为null的话则必然是位于Sync Queue，因为prev和next都是Sync Queue特有的域。那么那些“不为condition且node.prev不为null，但是node.next为null”的节点呢？这些节点有可能处于Sync Queue中（还未完全执行enq的时刻），这时就需要降级处理，从tail往前迭代以确保当前节点是否真的存在于Sync Queue中。

```java
private boolean findNodeFromTail(Node node) {
  // 从tail往前所有节点node
  Node t = tail;
  for (; ; ) {
    if (t == node)
      return true;
    if (t == null)
      return false;
    t = t.prev;
  }
}
```

回顾一下await()，在执行完isOnSyncQueue(Node)确认过后，如果Node仍未回到Sync Queue的话那么当前线程将会挂起，等待唤醒。

当前的调用顺序如下：

addConditionWaiter() -> fullyRelease(Node) -> isOnSyncQueue(Node) -> **checkInterruptWhileWaiting(Node)**

## checkInterruptWhileWaiting(Node) - 唤醒后检查是否有被中断

```java
private int checkInterruptWhileWaiting(Node node) {
  // 线程被唤醒后检测中断状态，如果为被中断就返回0，如果被中断了就返回非0
  return Thread.interrupted() ?
    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
  0;
}
```

线程被唤醒后检测中断状态，如果为被中断就返回0，如果被中断了就返回非0。在确认被中断之后的行为也会有所不同，继续深入到transferAfterCancelledWait(Node)看看。

### transferAfterCancelledWait(Node)

```java
final boolean transferAfterCancelledWait(Node node) {
  // 如果在signal()之前醒来则将原来的节点放会sync queue
  // 将当前在condition queue的节点重新入队至sync queue，nextWaiter的值在sync queue无影响，因为sync queue只会检测shared
  if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
    // enq是线程安全方法
    enq(node);
    return true;
  }
  /*
   * If we lost out to a signal(), then we can't proceed
   * until it finishes its enq().  Cancelling during an
   * incomplete transfer is both rare and transient, so just
   * spin.
   */
  // 如果在signal()之后醒来，则等待节点被放回sync queue
  // 可能会有线程竞争，也就是其他线程调用本方法的数据竞争，等待enq完成即可
  while (!isOnSyncQueue(node))
    // 最理想的情况下就是把时间片分给signal的那个线程
    Thread.yield();
  return false;
}
```

为什么有的选择抛出异常？有的却直接返回状态？我最开始也很疑惑，但是我在并发编程网上面看到Doug Lea在设计AQS时候的论文译文后就明白了。[译文](http://ifeve.com/aqs-2/)如下

> JSR133修订以后，就要求如果中断发生在`signal`操作之前，await方法必须在重新获取到锁后，抛出`InterruptedException`。但是，如果中断发生在`signal`后，`await`必须返回且不抛异常，同时设置线程的中断状态。

照着上述规范我们再来看这个方法，整个方法分成两块：if块和while块+return语句。上半部分属于**中断发生先于signal**，此时需归还节点至Sync Queue。下半部分属于**signal先于中断**，此时只能等待signal的入队操作完成。如果是**中断发生先于signal**那么要告知调用方抛出InterruptedException，反之则仅需标记中断。

好，await()的大部分相关方法已经击破，我们重新研究await()，意在将其一举拿下

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  // 添加至Condition Queue
  Node node = addConditionWaiter();
  // 在拥有exclusive锁的前提下，释放该锁，当后继节点唤醒时就算将当前拥有锁的线程逐出sync queue
  // 如果在sync queue中释放锁失败，就会将condition queue中的节点置为cancelled
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  // 等待新节点入队: 1.其他线程调用signal()会入队，2.当前线程被中断也会入队
  while (!isOnSyncQueue(node)) {
    // 挂起直至被signal
    LockSupport.park(this);
    // 如果是LockSupport唤醒的，会循环然后被挂起，等待进入到sync queue才退出循环
    // 但如果是强行中断，也会添加至sync queue，并且返回中断标识(非0)
    // 顾名思义，就是检测在park期间有没有被执行过中断操作
    // k1
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  // 节点自旋在获取锁成功后返回, 如果没有中断就返回false，如果被中断了就返回true
  if (//k2
    acquireQueued(node, savedState) && interruptMode != THROW_IE)
    // 这是执行到k1处都没有中断，但是在执行acquireQueued方法中却中断了的情况
    interruptMode = REINTERRUPT;
  // 如果在重新获得锁的期间有其他节点被添加到condition queue，就再整理一下
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

其实能从acquireQueued返回，无论是true or false都是说明获得了锁，接下来仅需重新整理一下队列，根据checkInterruptWhileWaiting和acquireQueued返回的中断情况决定是否应该抛出InterruptedException，随后即可回到线程调用await()的地方。至此await方法已经被我们拿下，其他await的衍生方法也是大同小异，这里便不再赘述。

终于到signal出场了。

## signal() - 唤醒Condition Queue中的首节点

```java
public final void signal() {
// 检测是否获取了排他锁
if (!isHeldExclusively())
throw new IllegalMonitorStateException();
Node first = firstWaiter;
// 如果队列不为null则释放第一个节点，可见condition queue是类似于公平队列
if (first != null)
doSignal(first);
}
```
需要注意的是isHeldExclusively()是需要子类去实现的，如果没有使用Condition那么就可以选择不实现，另一方面也可得知signal是需要持有锁才能调用的（与等待/通知的经典范式一致）。注意，signal选择的是最长等待时间的节点进行唤醒，我们看看其中doSignal(Node)的具体实现吧。

### doSignal(Node)

```java
private void doSignal(Node first) {
  do {
    // 将firstWaiter引用指向后面的节点，并且释放旧的firstWaiter的nextWaiter引用
    if ((firstWaiter = first.nextWaiter) == null)
      // 证明first是唯一一个节点，清空condition queue
      lastWaiter = null;
    // help GC
    first.nextWaiter = null;
  } while (!transferForSignal(first) &&
           // 直至condition queue没有节点或者已经将节点置入sync queue
           (first = firstWaiter) != null);
}
```

在do块中程序选取了最长等待时间的节点，直至transferForSignal成功或者Condition Queue已经没有更多的元素。如果你还记得await()中的transferAfterCancelledWait()方法，那么你肯定知道transferForSignal(Node)的作用必然是将Condition Queue的节点归还给Sync Queue。

### transferForSignal(Node) - 迁移节点并唤醒await的线程

```java
final boolean transferForSignal(Node node) {
  /*
   * If cannot change waitStatus, the node has been cancelled.
   * 说明该节点在释放锁的时候发生了异常，被修改为cancelled，返回false表示将传入的节点转移到sync queue失败
   */
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
    return false;

  /*
   * Splice onto queue and try to set waitStatus of predecessor to
   * indicate that thread is (probably) waiting. If cancelled or
   * attempt to set waitStatus fails, wake up to resync (in which
   * case the waitStatus can be transiently and harmlessly wrong).
   * 入队至sync queue，会修改Node的prev和next域，其中next域的修改延后于prev域
   */
  Node p = enq(node);
  // p节点就是node的直接前驱
  int ws = p.waitStatus;
  // 如果前驱节点为cancelled或者修改为signal失败，那么就唤醒node所封装的线程，会继续在k1处执行，进而退出循环执行AcquireQueued，将前驱节点更改为signal
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
    LockSupport.unpark(node.thread);
  // 如果前驱节点不为cancelled且成功将其更改为signal的话就不用麻烦acquireQueued进行自旋了
  return true;
}
```

如果将condition转换为0的CAS失败，说明该节点被置为cancelled了（详见awaitNanos(long)），返回false告知调用方signal()需要传入下一个节点。如果设置成功，则将节点重新放回Sync Queue，enq所返回的是旧的tail，也就是node的前驱。有意思的来了，我们分析一下if块，首先如果前驱是cancelled节点或者CAS设置signal失败的话（排他锁的状态仅可能有0、signal和cancelled），将会唤醒node节点的线程。这些线程的唤醒点在await()中while循环里的k1处，因为节点已经处于Sync Queue，所以退出while后将会调用acquireQueued(Node, int)**选择一个有效的前驱**，并通过CAS自旋修改其waitStatus。那么如果ws<=0而且成功将前驱的状态设置为signal的话则不需要再唤醒，因为**他的前驱就是一个有效的前驱**。

## signalAll() - 迁移Condition Queue所有节点

```java
public final void signalAll() {
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node first = firstWaiter;
  // condition queue不为null时
  if (first != null)
    doSignalAll(first);
}
```

与signal()的结构完全一致，唯一不同的就是调用doSignalAll(Node)而不是doSignal(Node)

```java
private void doSignalAll(Node first) {
  // 清空condition queue
  lastWaiter = firstWaiter = null;
  do {
    Node next = first.nextWaiter;
    first.nextWaiter = null;
    // 将condition queue的节点全部迁移至sync queue
    transferForSignal(first);
    first = next;
  } while (first != null);
}
```

本方法清空所有节点，全部迁移至Sync Queue，所以并不需要在乎transferForSignal的返回值。

## Condition Queue特点总结

- 必须在获得排他锁的前提下才能正确使用Condition
- isHeldExclusively()必须要被子类实现
- Condition Queue是一个**单向**链表
- Node.nextWaiter在Sync Queue被复用成节点类型(Exclusive、Shared)，因为Condition中都是Exclusive类型的节点，nextWaiter用作真正的本意 — 下一个waiter
- Condition Queue的节点类型waitStatus仅有0、condition和cancelled

## 后话

至此，关于Condition Queue的源码介绍已经完毕，感谢你的耐心阅读！