# eureka client从eureka server同步数据
## 概述
从eureka client端看：
eureka client定时从eureka server中拿到的数据放到自己的DiscoveryClient.LocalRegionApps/remoteRegionVsApps中，而后ribbon、feign等客户端组件使用DiscoveryClient.LocalRegionApps/remoteRegionVsApps查找最新的服务service，完成业务逻辑，如负载均衡

下面一起分析下eureka client是怎么从eureka server同步数据的
## eureka client从eureka server同步数据过程分析
入口
eureka-client-1.6.2.jar
DiscoveryClient.getAndStoreFullRegistry()方法代码如下
```
private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();
        Applications apps = null;
        // 使用jersey调用eureka server的接口获取数据
        EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
                ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            // 将获取到的数据放到this.localRegionApps属性中
            localRegionApps.set(this.filterAndShuffle(apps));
        }
}
```
eureka client从eureka server同步数据是定时同步的，所以DiscoveryClient.getAndStoreFullRegistry()方法是定时被调用的，调用栈如下
```
-DiscoveryClient.CacheRefreshThread.run()
--DiscoveryClient.fetchRegistry()
----DiscoveryClient.fetchRegistry(boolean forceFullRegistryFetch)
------DiscoveryClient.getAndStoreFullRegistry()
```

DiscoveryClient的内部类CacheRefreshThread是一个Runnable任务,既然是定时周期执行，那势必它会被其他线程池执行
```
class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}
```
执行这个任务的线程池,DiscoveryClient.initScheduledTasks(),看注释"Initializes all scheduled tasks"就知道，他不仅执行了这个任务，还有其他的任务在这里执行，代码逻辑如下
```
private void initScheduledTasks() {
    // 从eureka server拿到最新服务列表数据。这里有个开关，可以通过配置文件开或关: eureka.client.fetch-registry:true/false
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        scheduler.schedule(
                new TimedSupervisorTask(
                        "cacheRefresh",
                        scheduler,
                        cacheRefreshExecutor,
                        registryFetchIntervalSeconds,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new CacheRefreshThread()
                ),
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }
    
    // 是否注册自己到eureka server上。这里也有个开关，可以通过配置文件开或关: eureka.client.register-with-eureka:true/false
    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

        // Heartbeat timer
        scheduler.schedule(
                new TimedSupervisorTask(
                        "heartbeat",
                        scheduler,
                        heartbeatExecutor,
                        renewalIntervalInSecs,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new HeartbeatThread()
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS);

        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize

        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                        InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };

        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }

        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```
这里使用了3个线程池(一个周期性的，两个普通的)，和一个Runnable，完成各自的同步任务。