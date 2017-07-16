---
layout:     post
title:      "Spring Cloud Netflix - Eureka Server源码阅读"
subtitle:   "Understanding Eureka Peer to Peer communication"
author:     "Chris"
header-img: "img/post-bg-6.jpg"
tags:
    - Spring Cloud
    - 源码研读
---

![](http://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/images/aws_regions.png)

Eureka是一个基于REST并应用于AWS云服务的服务注册中心，并且经历过了Netflix公司的生产考验，绝对是我们值得细心研读的中间件。虽然我们可能并未接触过AWS，但在阅读Eureka之前应该简单地了解一下Amazon EC2中的某些概念，如[地区和可用区域](http://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)，这样能更好地理解Eureka中某些特定的术语。

在阅读本文之前，希望大家都已经对Eureka有简单的认识：

[Eureka REST operations](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

[Understanding eureka client server communication](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)

[Understanding Eureka Peer to Peer Communication](https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication)

## Register - 注册

Eureka Instance注册的REST入口在`com.netflix.eureka.resources.ApplicationResource#addInstance`

```java
    /**
     * Registers information about a particular instance for an
     * {@link com.netflix.discovery.shared.Application}.
     *
     * @param info
     *            {@link InstanceInfo} information of the instance.
     * @param isReplication
     *            a header parameter containing information whether this is
     *            replicated from other nodes.
     */
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
      	// *********** 字段校验 ************
        // validate that the instanceinfo contains all the necessary required fields
        if (isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else if (isBlank(info.getHostName())) {
            return Response.status(400).entity("Missing hostname").build();
        } else if (isBlank(info.getAppName())) {
            return Response.status(400).entity("Missing appName").build();
        } else if (!appName.equals(info.getAppName())) {
            return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
        } else if (info.getDataCenterInfo() == null) {
            return Response.status(400).entity("Missing dataCenterInfo").build();
        } else if (info.getDataCenterInfo().getName() == null) {
            return Response.status(400).entity("Missing dataCenterInfo Name").build();
        }
      
        // handle cases where clients may be registering with bad DataCenterInfo with missing data
        DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
      	// 仅当DataCenterInfo为AmazonInfo实例的时候，其父类有可能是UniqueIdentifier
        if (dataCenterInfo instanceof UniqueIdentifier) {
			// ......
        }
		// *********** 字段校验 END ************
        registry.register(info, "true".equals(isReplication));	// (1)
        return Response.status(204).build();  // 204 to be backwards compatible
    }
```

真正的注册操作在`(1)`处，需要注意的是`isReplication`变量取决于HTTP头`x-netflix-discovery-replication`的值。继续追踪`(1)`的调用栈，发现执行注册操作的方法是是`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#register` 

注意该方法的javadoc，他告诉了我们一个比较重要的讯息：将`InstanceInfo`实例信息注册到Eureka并且复制该信息到其他peer。如果当前收到的注册信息是来自其他peer的复制事件，那么将**不会将这个注册信息继续复制到其他peer**，这个标志位就是上面所述的`isReplication`。

```java
    /**
     * Registers the information about the {@link InstanceInfo} and replicates
     * this information to all peer eureka nodes. If this is replication event
     * from other replica nodes then it is not replicated.
     *
     * @param info
     *            the {@link InstanceInfo} to be registered and replicated.
     * @param isReplication
     *            true if this is a replication event from other replica nodes,
     *            false otherwise.
     */
    @Override
    public void register(final InstanceInfo info, final boolean isReplication) {
      	// 默认租约有效时长为90s
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
      	// 注册信息里包含则依照注册信息的租约时长
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
      	// super为AbstractInstanceRegistry
        super.register(info, leaseDuration, isReplication);
      	// 复制到其他peer
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }
```

我们看到是先获取到租约的有效时长，然后才是**真真正正地**委托给super执行注册操作`super.register(...)`并将注册信息复制到其他peer。register方法非常长，我们重点观察一下他的注册表的结构：

```java
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
```

该注册表是一个以app name为key（在Spring Cloud里就是`spring.application.name`），嵌套Map为value的ConcurrentHashMap结构。其嵌套Map是以Instance ID为key，Lease对象为value的键值结构。这个registry注册表在Eureka Server或SpringBoot Admin的监控面板上以Eureka Service这个角色出现。

```java
    /**
     * Registers a new instance with a given duration.
     *
     * @see com.netflix.eureka.lease.LeaseManager#register(java.lang.Object, int, boolean)
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
          	// 可以看出registry是一个以info的app name为key的Map结构, 也就是以spring.application.name的大写串为key
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
          	// registry的value的Map结构是以info的id为key，这里的id就是Eureka文档上的Instance ID，给你个例子你就想起是什么东西了：10.8.88.233:config-server:10888
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // .......
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
			// .......
        } finally {
            read.unlock();
        }
    }
```

上面是`register(...)`中关于registry的大致操作，其中有相当一部分的操作被略去了，如果感兴趣的话可以细致地研究一下。

## Renew and Cancel Lease - 续约与取消租约

续约的REST入口在`com.netflix.eureka.resources.InstanceResource#renewLease`

而取消租约的REST入口在`com.netflix.eureka.resources.InstanceResource#cancelLease`

两者的基本思想相似，经由`InstanceRegistry` -> `AbstractInstanceRegistry` -> `PeerAwareInstanceRegistryImpl`，其中PeerAwareInstanceRegistryImpl装饰了添加复制信息到其他节点的功能。其中register、renew、cancel、statusUpdate和deleteStatusOverride都会将其信息复制到其他节点。

## Fetch Registry - 获取注册信息

获取所有Eureka Instance的注册信息，`com.netflix.eureka.resources.ApplicationsResource#getContainers`，其注册信息由`ResponseCacheImpl`缓存，缓存的过期时间在其构造函数中由`EurekaServerConfig.getResponseCacheUpdateIntervalMs()`所控制，默认缓存时间为30s。而差量注册信息在Server端会保存得更为长一些（大约3分钟），因此获取的差量可能会重复返回相同的实例。Eureka Client会自动处理这些重复信息。

## Evcition

Eureke Server定期进行失效节点的清理，执行该任务的定时器被定义在`com.netflix.eureka.registry.AbstractInstanceRegistry#evictionTimer`，真正的任务是由他的内部类`AbstractInstanceRegistry#EvictionTask`所执行，默认为每60s执行一次清理任务，其执行间隔由`EurekaServerConfig#getEvictionIntervalTimerInMs`[**eureka.server.eviction-interval-timer-in-ms**]所决定。

回顾一下上面刚说完的注册流程，在`PeerAwareInstanceRegistryImpl#register`里面特别指出了**默认**的租约时长为90s[**eureka.Instance.lease-expiration-duration-in-seconds**]，即如果90s后都没有收到特定的Eureka Instance的Heartbeats，则会认为这个Instance已经失效（Instance在正常情况下**默认**每隔30s发送一个Heartbeats[**eureka.Instance.lease-renewal-interval-in-seconds**]，对以上两个默认值有疑问的可以翻阅`LeaseInfo`），`EvictionTask`则会把这个Instance纳入清理的范围。我们看看`EvictionTask`的清理代码是怎么写的。

```java
    public void evict(long additionalLeaseMs) {
        logger.debug("Running the evict task");

        if (!isLeaseExpirationEnabled()) {
            logger.debug("DS: lease expiration is currently disabled.");
            return;
        }

        // We collect first all expired items, to evict them in random order. For large eviction sets,
        // if we do not that, we might wipe out whole apps before self preservation kicks in. By randomizing it,
        // the impact should be evenly distributed across all applications.
      	// (2) 下面的for循环就是把registry中所有的Lease提取到局部变量expiredLeases
        List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
        for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
            Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
            if (leaseMap != null) {
                for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                    Lease<InstanceInfo> lease = leaseEntry.getValue();
                    if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                        expiredLeases.add(lease);
                    }
                }
            }
        }

        // To compensate for GC pauses or drifting local time, we need to use current registry size as a base for
        // triggering self-preservation. Without that we would wipe out full registry.
        int registrySize = (int) getLocalRegistrySize();
        int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());	// (3)
        int evictionLimit = registrySize - registrySizeThreshold;

        int toEvict = Math.min(expiredLeases.size(), evictionLimit);
        if (toEvict > 0) {
            logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

            Random random = new Random(System.currentTimeMillis());
            for (int i = 0; i < toEvict; i++) {
                // Pick a random item (Knuth shuffle algorithm)
                int next = i + random.nextInt(expiredLeases.size() - i);
                Collections.swap(expiredLeases, i, next);
                Lease<InstanceInfo> lease = expiredLeases.get(i);

                String appName = lease.getHolder().getAppName();
                String id = lease.getHolder().getId();
                EXPIRED.increment();
                logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
                internalCancel(appName, id, false);
            }
        }
    }
```

在`(2)`中把本地的registry中的租约信息全部提取出来，并在`(3)`通过`serverConfig.getRenewalPercentThreshold()`[**eureka.server.renewal-percent-threshold**，默认85%]计算出一个最大可剔除的阈值`evictionLimit`。

## 新增Peer Node时的初始化

在有多个Eureka Server的情况下，每个Eureka Server之间是如何发现对方的呢？

通过调试之后，我们根据调用链从下往上追溯，其初始入口为`org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#contextInitialized`

```java
	public void contextInitialized(ServletContext context) {
		try {
			initEurekaEnvironment();
			initEurekaServerContext();	// (4)

			context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
		}
		catch (Throwable e) {
			log.error("Cannot bootstrap eureka server :", e);
			throw new RuntimeException("Cannot bootstrap eureka server :", e);
		}
	}
```

由下个入口`(4)`最终可以定位到方法`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#syncUp`，从对应的javadoc上我们可以知道该方法从peer eureka节点往自己填充注册表信息。 如果操作失败则此同步操作将failover到其他节点，直到遍历完列表(service urls)为止。该方法与普通的Eureka Client注册到Eureka Server不同的一点是，其标志位`isReplication`为true，如果不记得这是什么作用的话可以翻阅到上面的`Register - 注册`小节。

## Peer Node信息的定时更新

首先我们看Eureka Server的上下文实体中的方法`com.netflix.eureka.DefaultEurekaServerContext#initialize`

```java
    @PostConstruct
    @Override
    public void initialize() throws Exception {
        logger.info("Initializing ...");
        peerEurekaNodes.start();	// (5)
        registry.init(peerEurekaNodes);
        logger.info("Initialized");
    }
```

该方法明确指出这是一个Spring Bean，在构建Bean完成后执行此方法，继续追踪`(5)`。

```java
    public void start() {
        taskExecutor = Executors.newSingleThreadScheduledExecutor(
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                        thread.setDaemon(true);
                        return thread;
                    }
                }
        );
        try {
            updatePeerEurekaNodes(resolvePeerUrls());	// (6)
            Runnable peersUpdateTask = new Runnable() {	// (7)
                @Override
                public void run() {
                    try {
                        updatePeerEurekaNodes(resolvePeerUrls());	// (6)
                    } catch (Throwable e) {
                        logger.error("Cannot update the replica Nodes", e);
                    }

                }
            };
            taskExecutor.scheduleWithFixedDelay(
                    peersUpdateTask,	// (7)
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),	// (8)
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    TimeUnit.MILLISECONDS
            );
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
        for (PeerEurekaNode node : peerEurekaNodes) {
            logger.info("Replica node URL:  " + node.getServiceUrl());
        }
    }
```

上面这段代码很清晰地告诉我们在启动Eureka Server的时候就会调用`updatePeerEurekaNodes(...)`更新peer的状态，并封装为一个Runnable进行周期性更新。这个定时时间由`serverConfig.getPeerEurekaNodesUpdateIntervalMs()`[**eureka.server.peer-eureka-nodes-update-interval-ms**]所控制，默认值为600s，即10min。一直经由`EndpointUtils#getDiscoveryServiceUrls`、`EndpointUtils#getServiceUrlsFromConfig`至`EurekaClientConfigBean#getEurekaServerServiceUrls`获得对应zone的service urls，如有需要可以覆盖上述`getEurekaServerServiceUrls`方法以动态获取service urls，而不是选择Spring Cloud默认从properties文件读取。

## Self Preservation - 自我保护

当新增Eureka Server时，他会先尝试从其他Peer上获取所有Eureka Instance的注册信息。如果在获取时出现问题，该Eureka Server会在放弃之前尝试在其他Peer上获取注册信息。如果这个Eureka Server成功获取到所有Instance的注册信息，那么他就会根据所获取到的注册信息设置应该接收到的续约阈值。如果在任何时候续约的阈值低于所设定的值（在15分钟[**eureka.server.renewal-threshold-update-interval-ms**]内低于85%[**eureka.server.renewal-percent-threshold**]），则该Eureka Server会出于保护当前注册列表的目的而停止将任何Instance进行过期处理。

在Netflix中上述保护措施被成为自我保护模式，主要是用于Eureka Server与Eureka Client存在网络分区情况下的场景。在这种情况下，Eureka Server尝试保护其已有的实例信息，但如果出现大规模的网络分区时，相应的Eureka Client会获取到大量无法响应的服务。所以，Eureka Client必须确保对于一些不存在或者无法响应的Eureka Instance具备更加弹性的应对策略，例如快速超时并尝试其他实例。

在网络分区出现时可能会发生以下几种情况：

- Peer之间的心跳可能会失败，某Eureka Server检测到这种情况并为了保护当前的注册列表而进入了自我保护模式。新的注册可能发生在某些**孤立的Eureka Server**上，某些Eureka Client可能会拥有新的注册列表，而另外一些则可能没有（不同的实例视图）。
- 当网络恢复到稳定状态后，Eureka Server会进行自我修复。当Peer能正常通信之后注册信息会被重新同步。
- 最重要的一点是，在网络中断期间Eureka Server应该更距弹性，但在这段期间Eureka Client可能会有不同的实例视图。