---
layout:     post
title:      "Apache Curator - 选主与分布式锁"
subtitle:   "Leader Election and Shared Reentrant Lock"
author:     "Chris"
header-img: "img/post-bg-3.jpg"
tags:
    - ZooKeeper
---

## Leader Election

在分布式计算中，leader election是指定一个任务的组织者在几台计算机（节点）之间分配的过程。 在任务开始之前，所有网络节点都不知道哪个节点将作为任务的“领导者”或协调者。 然而，在执行了leader election算法之后，整个网络中的每个节点识别出一个特定的唯一节点作为任务负责人。

Curator的[相关文档](http://curator.apache.org/curator-recipes/leader-election.html)中强烈建议我们实现LeaderSelectorListenerAdapter接口，以应对connection loss和session expired的情况。对于以上的两种状态都会抛出CancelLeadershipException，使得LeaderSelector实例尝试去中断和cancel正在执行`takeLeadership`方法的线程。

下面是5个节点抢夺leader的示例：

```java
/**
 * @author Zhao Junjian
 */
public class LeaderElectionTest {
    private static final Random RANDOM = new SecureRandom();
    private static final String LEADER_PATH = "/prontera/leader";
    private static final String CONNECTION_STR = "localhost:2181,localhost:2182,localhost:2183";

    public static void main(String[] args) throws IOException {
        final int nThreads = 5;
        final ExecutorService pool = Executors.newFixedThreadPool(nThreads);
        final Deque<CuratorFramework> clientList = Lists.newLinkedList();
        final Deque<LeaderCompetition> selectorList = Lists.newLinkedList();
        try {
            for (int i = 0; i < nThreads; i++) {
                int finalI = i;
                pool.execute(() -> {
                    final CuratorFramework client = CuratorFrameworkFactory.newClient(CONNECTION_STR, new ExponentialBackoffRetry(800, 3));
                    final LeaderCompetition selector = new LeaderCompetition(client, LEADER_PATH, "Client#" + finalI);
                    clientList.addLast(client);
                    selectorList.addLast(selector);
                    client.start();
                    selector.start();
                });
            }
            // 输入空行结束，一定要在main方法中才有效
            new BufferedReader(new InputStreamReader(System.in)).readLine();
        } finally {
            for (LeaderCompetition s : selectorList) {
                CloseableUtils.closeQuietly(s);
            }
            for (CuratorFramework c : clientList) {
                CloseableUtils.closeQuietly(c);
            }
            pool.shutdown();
        }
    }

    private static final class LeaderCompetition extends LeaderSelectorListenerAdapter implements Closeable {
        /**
         * selector's name
         */
        private final String name;

        /**
         * selector instance
         */
        private final LeaderSelector selector;

        private LeaderCompetition(CuratorFramework client, String leaderPath, String name) {
            this.name = name;
            selector = new LeaderSelector(client, leaderPath, this);
            // 行使完leader的权力之后再次参与竞选
            selector.autoRequeue();
        }

        public void start() {
            selector.start();
        }

        /**
         * 注意这是{@link Closeable}的方法
         */
        @Override
        public void close() throws IOException {
            selector.close();
        }

        /**
         * 只要从该方法退出，则说明放弃了leader的权力，并重新进行选举
         */
        @Override
        public void takeLeadership(CuratorFramework client) throws Exception {
            try {
                System.out.println(name + " has acquired leadership.");
                TimeUnit.MILLISECONDS.sleep(700);
                // throw new CancelLeadershipException(name);
                // KillSession.kill(client.getZookeeperClient().getZooKeeper(), LEADER_PATH);
                // System.out.println(name + " session过期");
                System.out.println(name + " print " + UUID.randomUUID());
                //TimeUnit.MILLISECONDS.sleep(700);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    }

}
```

## Shared Reentrant Lock

完全的分布式锁是全局同步的，意味着在任何一个时刻不会有两个客户端拥有同一把锁。

对于分布式锁我们需要考虑的一点是，如果在获取锁成功后且在释放锁之前，因为各种各样的原因，导致session expired从而使得该节点在毫不知情的情况下失去了锁，那么此时该怎么处理呢？在获取锁后出现的connection loss和session expired的情况都直接如leader election采用无差别抛出异常的处理方式。

我们划分一下：

* 如果是计算类的逻辑，那么直接舍弃其结果，不予返回。
* 如果是需要操作数据库但并不涉及调用其他服务接口的，抛出异常使得Spring自动回滚该事务。
* 如果是需要操作数据库并且调用了其他服务接口的，不仅需要回滚还需要接口补偿。

```java
/**
 * @author Zhao Junjian
 */
public class LockTest {

    private static final String LOCK_PATH = "/prontera/lock";
    private static final String CONNECTION_STR = "localhost:2181,localhost:2182,localhost:2183";

    @Test
    public void testInterProcessMutex() throws Exception {
        //int qty = 10;
        int qty = 1;
        int loop = qty * 100;
        final ExecutorService pool = Executors.newFixedThreadPool(qty);
        for (int i = 0; i < qty; i++) {
            int finalI = i;
            pool.execute(() -> {
                CuratorFramework client = null;
                try {
                    client = CuratorFrameworkFactory.newClient(CONNECTION_STR, 2000, 1000, new ExponentialBackoffRetry(800, 3));
                    client.start();
                    client.blockUntilConnected();
                    final MutexWork work = new MutexWork(client, LOCK_PATH, "Client#" + finalI);
                    for (int k = 0; k < loop; k++) {
                        work.execute();
                    }
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                    e.printStackTrace();
                } finally {
                    if (client != null) {
                        CloseableUtils.closeQuietly(client);
                    }
                }
            });
        }
        pool.shutdown();
        pool.awaitTermination(15, TimeUnit.MINUTES);
    }

    private static final class MutexWork implements ConnectionStateListener {
        private final String name;
        private final CuratorFramework client;
        private final InterProcessMutex lock;
        private boolean shouldRollback;

        private MutexWork(CuratorFramework client, String lockPath, String name) {
            this.name = name;
            this.client = client;
            this.lock = new InterProcessMutex(client, lockPath);
            client.getConnectionStateListenable().addListener(this);
        }

        /**
         * 在获取锁后出现的connection loss和session expired的情况都直接如leader election采用无差别抛出异常的处理方式。
         * 如果是计算类的逻辑，那么直接舍弃其结果，不予返回。
         * 如果是需要操作数据库但并不涉及调用其他服务接口的，抛出异常使得Spring自动回滚该事务
         * 如果是需要操作数据库并且调用了其他服务接口的，不仅需要回滚还需要接口补偿
         */
        //@org.springframework.transaction.annotation.Transactional(rollbackFor = Exception.class)
        public void execute() throws Exception {
            lock.acquire();
            try {
                System.out.println(name + " has acquired the lock. ");
                //TimeUnit.MILLISECONDS.sleep(500);
                // 人为地使session过期，注意此时会自动释放锁，在下个线程处理的时候会出现not mutex
                KillSession.kill(client.getZookeeperClient().getZooKeeper(), CONNECTION_STR);
                System.out.println(name + " session had been killed causing lose the lock.");
                // 如果session过期或是connection出现问题则即可抛出异常
                throwIfSuspendedOrLost();
                // ********* 业务逻辑 begin ************
                TimeUnit.MILLISECONDS.sleep(500);
                // ********* 业务逻辑 end ************
                throwIfSuspendedOrLost();
            } finally {
                TimeUnit.MILLISECONDS.sleep(500);
                System.out.println(name + " is releasing if it has a lock.");
                lock.release();
            }
        }

        private void throwIfSuspendedOrLost() throws Exception {
            // 旨在使curator马上更新状态
            client.checkExists().forPath(LOCK_PATH);
            if (shouldRollback) {
                // 使得Spring管理的事务回滚
                throw new IllegalStateException("roll back");
            }
        }

        @Override
        public void stateChanged(CuratorFramework client, ConnectionState newState) {
            /**
             It is strongly recommended that you add a ConnectionStateListener and watch for SUSPENDED and LOST state changes.
             If a SUSPENDED state is reported you cannot be certain that you still hold the lock unless you subsequently receive a RECONNECTED state.
             If a LOST state is reported it is certain that you no longer hold the lock.

             强烈建议添加ConnectionStateListener并监听SUSPENDED和LOST事件。
             如果SUSPENDED状态发生，此时无法确认是否还持有锁，直至后续收到RECONNECTED事件
             如果LOST状态，那么此时肯定不再持有锁了

             如果是session expired，那么状态将会是SUSPENDED->LOST->RECONNECTED
             如果是connection loss，那么状态将会是SUSPENDED->RECONNECTED

             */
            System.err.println(name + ": " + newState);
            if (newState == ConnectionState.SUSPENDED || newState == ConnectionState.LOST) {
                shouldRollback = true;
                //throw new IllegalStateException(name + " session expired: " + newState);
            }
        }
    }
}
```