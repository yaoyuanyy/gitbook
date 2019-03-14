# zuul1_ribbon负载均衡源码分析

ribbon的核心类是ILoadBalancer和IRule，ILoadBalancer默认的实现类为：ZoneAwareLoadBalancer；
IRule默认的实现类为：springcloudribbon是ZoneAvoidanceRule,代码在RibbonClientConfiguration.ribbonRule(IClientConfig)中
而ribbon本身的默认实现为RoundRobinRule,代码在BaseLoadBalancer.setRule(IRule)中

## 项目启动时
### 地方


## 访问url时
例： http://localhost:5555/api-a/add?a=1&b=2&accessToken=12


## ribbon定时从eureka client拿到最新服务列表
先总结下：从哪到哪->从DiscoveryClient.LocalRegionApps/remoteRegionVsApps获取服务列表数据塞到BaseLoadBalacer.upServerList

注意：而eureka client其实是从eureka server中拿到的数据放到自己的DiscoveryClient.LocalRegionApps/remoteRegionVsApps中，详解参见《2019-03-11_springcloud_eureka_client从server同步数据》

调用栈
-PollingServerListUpdater.start(UpdateAction)
--ServerListUpdater.UpdateAction.doUpdate()
----DynamicServerListLoadBalancer.updateListOfServers(
-----DiscoveryEnabledNIWSServerList.getUpdatedListOfServers() // 获取到servers
------this.obtainServersViaDiscovery()
-------DiscoverClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion)
-----ZonePreferenceServerListFilter.getFilteredListOfServers(servers)
-----DynamicServerListLoadBalancer.updateAllServerList(servers)
------DynamicServerListLoadBalancer.setServersList(servers)// servers赋值到this.super.upServerList

调用栈总结：从DiscoveryClient.LocalRegionApps/remoteRegionVsApps获取服务列表数据塞到BaseLoadBalacer.upServerList的同时，DynamicServerListLoadBalancer.setServersList()方法还会将zone和servers作映射形成map，赋值给LoadBalancerStats.upServerListZoneMap和zoneStatsMap，留作后用

一个zone-->一个BaseLoadBalancer-->一个IRule,其中一个BaseLoadBalancer包含upServerList和allServerList，即服务集合


入口
PollingServerListUpdater.start(UpdateAction)方法
```
@Override
public synchronized void start(final UpdateAction updateAction) {
    if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                try {
                    updateAction.doUpdate(); // 更新服务列表的任务
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };
        
        // 使用线程池周期性的执行wrapperRunnable任务
        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                wrapperRunnable, //任务
                initialDelayMs, // 1s
                refreshIntervalMs, // 30s
                TimeUnit.MILLISECONDS
        );
    }
}  
```

DynamicServerListLoadBalancer.updateListOfServers()代码逻辑
```
@VisibleForTesting
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);
        }
    }
    updateAllServerList(servers);
}
```
serverListImpl.getUpdatedListOfServers()会调用DiscoveryEnabledNIWSServerList.obtainServersViaDiscovery()方法获取servers集合

DiscoveryEnabledNIWSServerList.obtainServersViaDiscovery()方法
```
private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
        EurekaClient eurekaClient = eurekaClientProvider.get();
        if (vipAddresses!=null){
            for (String vipAddress : vipAddresses.split(",")) {
                // if targetRegion is null, it will be interpreted as the same region of client
                List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
                for (InstanceInfo ii : listOfInstanceInfo) {
                    if (ii.getStatus().equals(InstanceStatus.UP)) {

                        if(shouldUseOverridePort){
                            InstanceInfo copy = new InstanceInfo(ii);
                            DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure, shouldUseIpAddr);
                        des.setZone(DiscoveryClient.getZone(ii));
                        serverList.add(des);
                    }
                }
            }
        }
        return serverList;
    }
```
从代码中可以看到，listOfInstanceInfo持有从DiscoveryClient.LocalRegionApps/remoteRegionVsApps获取到的信息后，与region和zone结合形成DiscoveryEnabledServer实例，流入到List<DiscoveryEnabledServer>集合返回

```
@Override
public void setServersList(List lsrv) {
    // 赋值给BaseLoadBalacer.upServerList
    super.setServersList(lsrv);
    List<T> serverList = (List<T>) lsrv;
    Map<String, List<Server>> serversInZones = new HashMap<String, List<Server>>();
    for (Server server : serverList) {
        // make sure ServerStats is created to avoid creating them on hot
        // path
        // 赋值给LoadBalancerStats.zoneStatsMap
        getLoadBalancerStats().getSingleServerStat(server);
        String zone = server.getZone();
        if (zone != null) {
            zone = zone.toLowerCase();
            List<Server> servers = serversInZones.get(zone);
            if (servers == null) {
                servers = new ArrayList<Server>();
                serversInZones.put(zone, servers);
            }
            servers.add(server);
        }
    }
    // 
    setServerListForZones(serversInZones);
}
```



轮询规则实现
AbstractServerPredicate class
```
private int incrementAndGetModulo(int modulo) {
    for (;;) {
        int current = nextIndex.get();
        int next = (current + 1) % modulo;
        if (nextIndex.compareAndSet(current, next) && current < modulo)
            return current;
    }
}
```

