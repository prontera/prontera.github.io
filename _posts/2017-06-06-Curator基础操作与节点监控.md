---
layout:     post
title:      "Apache Curator - 基础技法与节点监控"
subtitle:   "CRUD and Cache"
author:     "Chris"
header-img: "img/post-bg-2.jpg"
tags:
    - ZooKeeper
---

## 前言

Apache Curator是一个针对于[Apache ZooKeeper](https://zookeeper.apache.org/)的分布式协调服务的Java库，使得操作Apache ZooKeeper变得更为简单和可靠。其中更包括了针对于常见使用场景的高层次封装，如分布式锁，服务发现与注册和选主等。

目前Curator的release版本有2.x.x和3.x.x，其中Curator 2.x.x是兼容ZooKeeper 3.4.x和ZooKeeper 3.5.x，而Curator 3.x.x仅兼容ZooKeeper 3.5.x。由于截至编写该博客的时候Dokcer Hub上也没有官方发布的ZooKeeper 3.5.x的镜像，所以该文使用Curator 2.x.x作为工具库。

搭建ZooKeeper集群的compose如下

```dockerfile
version: '2'
services:
    zoo1:
        image: zookeeper:3.4
        restart: always
        ports:
            - 2181:2181
        environment:
            ZOO_MY_ID: 1
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

    zoo2:
        image: zookeeper:3.4
        restart: always
        ports:
            - 2182:2181
        environment:
            ZOO_MY_ID: 2
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

    zoo3:
        image: zookeeper:3.4
        restart: always
        ports:
            - 2183:2181
        environment:
            ZOO_MY_ID: 3
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
```

## CRUD示例

### 测试前的准备

```java
private CuratorFramework client;
private static final String PARENT_PATH = "/prontera";
private static final String CONNECT_STRING = "localhost:2181,localhost:2182,localhost:2183";

@Before
public void setUp() throws Exception {
  client = CuratorFrameworkFactory.newClient(CONNECT_STRING, new ExponentialBackoffRetry(800, 3));
}
```

### 节点的创建、读取与删除

创建节点时`withMode(CreateMode.PERSISTENT)`是默认选项，如果需要创建的正是persistence类型节点，那么该语句是可以被忽略。

我们来假设这样一个edge case：在获得ZooKeeper的连接后，在执行真正的创建节点操作前发生session expired，此时Curator应该怎么做？我们先查看一下Curator的[Connection Guarantees](http://curator.apache.org/errors.html)

> Curator Framework不断监控着ZooKeeper的连接。进一步地说，每一个操作都被重试机制所包装。因此可以做出以下的保证：
>
> - 每个Curator的操作都会等待至ZooKeeper的连接被成功建立。
> - 每个Curator的操作 (create, getData, etc.) 都会根据当前所设定的重试机制去管理connection loss和session expiration的情况。
> - 如果connection暂时丢失，Curator将会根据所设定的重试机制进行重试，直至成功。
> - 所有Curator Recipes都会尝试使用一种合适的方式去处理connection问题。

这里使用Curator自带的工具类KillSession模拟session expired的状态

```java
    /**
     * 如果路径上的<tt>父节点</tt>不存在，默认抛出{@link org.apache.zookeeper.KeeperException.NoNodeException}
     * 如果创建的节点存在，则抛出{@link org.apache.zookeeper.KeeperException.NodeExistsException}
     */
    @Test
    public void testCreatePersistenceButKillSession() throws Exception {
        client.start();
        // client.blockUntilConnected();
        final String path = PARENT_PATH + "/persist/session";
        final String val = "hello";
        // kill session
        KillSession.kill(client.getZookeeperClient().getZooKeeper(), CONNECT_STRING);
        System.out.println("manual session expired.");
        // withMode(CreateMode.PERSISTENT)是多余的，默认就是persist类型节点
        // creatingParentsIfNeeded()，如果父路径不存在则自动创建
        client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path, val.getBytes(Charsets.UTF_8));
        System.out.println("create node " + path + " successfully");
        final byte[] result = client.getData().forPath(path);
        client.delete().guaranteed().forPath(path);
        client.close();
        Assert.assertArrayEquals(val.getBytes(Charsets.UTF_8), result);
    }

```

该测试单元的结果为passed，Curator通过重试正确地处理了session expired的情况。如果有细心看可以发现清理现场的工作中我们调用了`client.delete().guaranteed().forPath(path)`，这个guaranteed()将会在下文详述。

### 节点的更新

本示例依然模拟了session expired的情况，Curator依然能应对自如。

```java
    @Test
    public void testUpdateButKillSession() throws Exception {
        client.start();
        final String path = PARENT_PATH + "/persist/update/session";
        client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path, "hello".getBytes(Charsets.UTF_8));
        System.out.println("创建节点 " + path + " 成功");
        byte[] result = client.getData().forPath(path);
        System.out.println("更新前的值为 " + new String(result, Charsets.UTF_8));
        // delete
        KillSession.kill(client.getZookeeperClient().getZooKeeper(), CONNECT_STRING);
        System.out.println("manual session expired before updating.");
        client.setData().forPath(path, "java".getBytes(Charsets.UTF_8));
        result = client.getData().forPath(path);
        System.out.println("更新后的值为 " + new String(result, Charsets.UTF_8));
        client.delete().guaranteed().forPath(path);
        client.close();
        Assert.assertArrayEquals("java".getBytes(Charsets.UTF_8), result);
    }
```

### Guarantee Delete

```java
client.delete().guaranteed().forPath(path);
```

此机制是为了解决这样一种情况：删除一个节点时可能会由于connection问题而失败。
再进一步，如果待删除的节点是"临时"的，那么该节点就不会被自动删除，因为当前的session依然是有效的。
这对于分布式锁来说是不能接受的。
当guaranteed机制被设置，Curator将会记录删除失败的节点，并尝试在后台去删除直至成功。
注意：在删除失败的时候依然会抛出异常。但是，可以保证只要CuratorFramework实例还存在的时候就会去尝试删除该节点。

### Create With Protection

```java
client.create().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, payload);
```

在创建"顺序临时"节点时有一种情况。创建操作在服务端成功，但是服务端在返回节点名称给客户端时crash了。然后，当前的zk session依然是有效的，所以这个"临时节点"也不会删除。所以对于客户端来说这样就没办法得知哪个"顺序且临时"的节点是由自己创建的了。
即便不是"顺序且临时"的节点，对于成功在服务端创建但是客户端无法得知的情况是会存在的。对于上述的情况，使用create builder中的protection模式可以解决。节点的名称会添加一个GUID的前缀。当节点创建失败的时候将会触发重试机制。在重试当中，他的父路径会搜索这个GUID前缀的节点。如果节点存在，则可能是lost node，此时服务端将返回该节点。

## Cache

Curator中的Cache可以作为监听器使用，PathChildrenCache监听指定节点的子节点的CUD事件，NodeCache监听指定节点的CUD操作，而TreeCache则是PathChildrenCache与NodeCache的结合体。

### 测试前的准备

```java
private CuratorFramework client;
private static final String PARENT_PATH = "/prontera";

@Before
public void setUp() throws Exception {
  String connectString = "localhost:2181,localhost:2182,localhost:2183";
  client = CuratorFrameworkFactory.newClient(connectString, new ExponentialBackoffRetry(800, 3));
}
```

### PathChildrenCache

A Path Cache is used to watch a ZNode. Whenever a child is added, updated or removed, the Path Cache will change its state to contain the current set of children, the children's data and the children's state. Path caches in the Curator Framework are provided by the PathChildrenCache class. Changes to the path are passed to registered PathChildrenCacheListener instances.

```java
    @Test
    public void testPathCache() throws Exception {
        client.start();
        // 用于监控一个znode的子节点的CUD，并且能获取到最新的子节点数据和子节点状态(Stat)
        final PathChildrenCache cache = new PathChildrenCache(client, PARENT_PATH + "/path-cache", true);
        try {
            final PathChildrenCacheListener listener = getPathCacheListener();
            cache.getListenable().addListener(listener);
            cache.start();
            // 接着就在zkCli处操作，观察当前的控制台打印的信息
            TimeUnit.SECONDS.sleep(Long.MAX_VALUE);
        } finally {
            CloseableUtils.closeQuietly(cache);
            CloseableUtils.closeQuietly(client);
        }
    }

    private PathChildrenCacheListener getPathCacheListener() {
        return new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework client1, PathChildrenCacheEvent event) throws Exception {
                final ChildData data = event.getData();
                String path = "null";
                String val = "null";
                if (data != null) {
                    path = data.getPath();
                    val = new String(data.getData(), Charsets.UTF_8);
                }
                System.out.println(event.getType() + ": " + path + " " + val);
            }
        };
    }
```

### NodeCache

A utility that attempts to keep the data from a node locally cached. This class will watch the node, respond to update/create/delete events, pull down the data, etc. You can register a listener that will get notified when changes occur.

```java
    @Test
    public void testNodeCache() throws Exception {
        client.start();
        // 可以观察到指定节点的CUD
        final NodeCache cache = new NodeCache(client, PARENT_PATH + "/node-cache");
        try {
            final NodeCacheListener listener = new NodeCacheListener() {
                @Override
                public void nodeChanged() throws Exception {
                    // 在面对删除事件的时候getCurrentData会返回null
                    if (cache.getCurrentData() != null) {
                        System.out.println(new String(cache.getCurrentData().getData(), Charsets.UTF_8));
                    } else {
                        System.out.println("the node had beed deleted.");
                    }
                }
            };
            cache.getListenable().addListener(listener);
            cache.start();
            // 接着就在zkCli处操作，观察当前的控制台打印的信息
            TimeUnit.SECONDS.sleep(Long.MAX_VALUE);
        } finally {
            CloseableUtils.closeQuietly(cache);
            CloseableUtils.closeQuietly(client);
        }
    }
```

### TreeCache

A utility that attempts to keep all data from all children of a ZK path locally cached. This class will watch the ZK path, respond to update/create/delete events, pull down the data, etc. You can register a listener that will get notified when changes occur.

```java
    @Test
    public void testTreeCache() throws Exception {
        client.start();
        // Path Cache + Node Cache，该节点与其子节点的添加和移除都用相同的事件，如NODE_ADDED, NODE_UPDATED和NODE_REMOVED
        final TreeCache cache = new TreeCache(client, PARENT_PATH + "/tree-cache");
        try {
            final TreeCacheListener listener = getTreeCacheListener();
            cache.getListenable().addListener(listener);
            cache.start();
            // 接着就在zkCli处操作，观察当前的控制台打印的信息
            TimeUnit.SECONDS.sleep(Long.MAX_VALUE);
        } finally {
            CloseableUtils.closeQuietly(cache);
            CloseableUtils.closeQuietly(client);
        }
    }

    private TreeCacheListener getTreeCacheListener() {
        return new TreeCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
                final ChildData data = event.getData();
                String path = "null";
                String val = "null";
                if (data != null) {
                    path = data.getPath();
                    val = new String(data.getData(), Charsets.UTF_8);
                }
                System.out.println(event.getType() + ": " + path + " " + val);
            }
        };
    }
```

