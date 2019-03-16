# springcloud-ribbon负载均衡源码分析
本篇内容特别适合对ribbon有基本了解的同学，初学者可以看运行起ribbon的demo后再来这篇，效率更高。所以ribbon的理论功能和介绍不作阐述，只从源码角度关注细节。

## 前言
适应版本：springcloud1和springcloud2

本篇是ribbon结合eureka实现负载均衡的源码解析，学完本篇后，将清楚以下知识点：
- ribbon实现负载均衡需要的server列表是如何获取和定时同步的，获取到的server列表存放在哪里
- ribbon优先选择同zone的server实现细节
- 当没有同zone的server时，ribbon使用哪些服务server完成请求
- ribbon从同/不同zone的server列表确定一个server的算法

我们知道，eureka server是动态注册服务列表的，当有服务出现故障或下线时，eureka client会更新服务列表，以便eureka client获取最新的server列表。ribbon作为一个eureka client，又是一个consumer组件，从功能上讲他有个两方面的功能
1. 定期从eureka server获取最新server列表，见[eureka client从server同步数据](https://yaoyuanyy.github.io/gitbook/gitbook_web/spring%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97/2019-03-11_springcloud_eureka_client%E4%BB%8Eserver%E5%90%8C%E6%AD%A5%E6%95%B0%E6%8D%AE.html)
2. ribbon定时从eureka client拿到最新server列表。当一个请求过来时从即server列表负载均衡确定一个server，完成request请求


本篇主要分析当一个请求过来时，ribbon从server列表负载均衡确定一个server的过程，包括从哪里获取的server列表，怎样确定一个server的。下面进入正文


## ribbon核心类
ribbon进行负载均衡功能有两个核心类: ILoadBalancer和IRule，ILoadBalancer默认的实现类为：ZoneAwareLoadBalancer；
IRule默认的实现类为：springcloudribbon是ZoneAvoidanceRule,代码在RibbonClientConfiguration.ribbonRule(IClientConfig)中
而ribbon本身的默认实现为RoundRobinRule,代码在BaseLoadBalancer.setRule(IRule)中




## 核心类实例化和初始化
### 项目启动时涉及ribbon的XXXConfiguration.class有两个: RibbonClientConfiguration.class和EurekaRibbonClientConfiguration.class.
RibbonClientConfiguration负责ILoadBalancer和IRule的实例化和初始化；
```
@Configuration
@EnableConfigurationProperties
@Import({HttpClientConfiguration.class, OkHttpRibbonConfiguration.class, RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class})
public class RibbonClientConfiguration {
    ...
}
```
EurekaRibbonClientConfiguration负责seviceId, zone, server instance相关的初始化工作
```
@Configuration
public class EurekaRibbonClientConfiguration {
    ...
}
```



## 一个request请求过来
当在浏览器访问url时，例: http://localhost:5555/api-a/add?a=1&b=2&accessToken=12  整个负载均衡过程就开始了，如下


### ribbon周期性从eureka client拿到最新服务列表
- 先总结下：从哪到哪->从DiscoveryClient.LocalRegionApps/remoteRegionVsApps获取服务列表数据塞到BaseLoadBalacer.upServerList

注意：而ribbon(eureka client)的DiscoveryClient.LocalRegionApps/remoteRegionVsApps数据是先周期性从eureka server中拿到的，详解参见[eureka client从server同步数据](https://yaoyuanyy.github.io/gitbook/gitbook_web/spring%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97/2019-03-11_springcloud_eureka_client%E4%BB%8Eserver%E5%90%8C%E6%AD%A5%E6%95%B0%E6%8D%AE.html)

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



### 轮询规则实现
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

