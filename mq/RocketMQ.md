# RocketMQ

## RocketMQ Architecture

![Architecture](http://rocketmq.apache.org/assets/images/rmq-basic-arc.png)

### NameServer

#### NameServer启动流程

`org.apache.rocketmq.namesrv.NamesrvStartup`

```java
public static NamesrvController main0(String[] args) {

    try {
        //创建
        NamesrvController controller = createNamesrvController(args);
        //启动
        start(controller);
        //输出日志
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}

```

1.创建

`org.apache.rocketmq.namesrv.NamesrvStartup#createNamesrvController`

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
    ...
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(9876);
    
    ...
        
    final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

    // remember all configs to prevent discard
    controller.getConfiguration().registerConfig(properties);

    return controller;
}
```

`org.apache.rocketmq.namesrv.NamesrvController`

```java
public class NamesrvController {
    private final NamesrvConfig namesrvConfig;
    private final NettyServerConfig nettyServerConfig;
    private final ScheduledExecutorService scheduledExecutorService 
    private final KVConfigManager kvConfigManager;
    private final RouteInfoManager routeInfoManager;
    
    private RemotingServer remotingServer;
    private BrokerHousekeepingService brokerHousekeepingService;
    private ExecutorService remotingExecutor;
    private Configuration configuration;
    private FileWatchService fileWatchService;
}
```

`org.apache.rocketmq.common.namesrv.NamesrvConfig`

```properties
#rocketmq主目录
rocketmqHome
#存储KV配置属性的持久化路径
kvConfigPath
#默认配置文件路径，不生效
configStorePath
#是否支持顺序消息，默认不支持
orderMessageEnable
productEnvName
clusterTest
```

`org.apache.rocketmq.remoting.netty.NettyServerConfig`

```properties
#监听端口，默认会被初始化为9876
listenPort
#业务线程池线程个数
serverWorkerThreads
#Netty public 任务线程池线程个数，Netty网络设计，根据业务类型会创建不同的线程池，处理消息发送、消息消费、心跳检测。业务类型未注册线程池，由public线程执行
serverCallbackExecutorThreads
#IO线程池个数，主要是NameServer、Broker端解析请求、返回相应的线程个数。主要处理网络请求，解析请求包，然后转发到各个业务线程池完成具体的业务操作，将结果返回给调用方
serverSelectorThreads
#send oneway 消息请求并发度（Broker端参数）
serverOnewaySemaphoreValue
#异步消息发送最大并发度（Broker端参数）
serverAsyncSemaphoreValue
#网络连接最大空闲时间，默认120S
serverChannelMaxIdleTimeSeconds
#网络socket发送缓存区大小
serverSocketSndBufSize
#网络socket接受缓存区大小
serverSocketRcvBufSize
#是否开启缓存
serverPooledByteBufAllocatorEnable
#是否启用EpollIO模型
useEpollNativeSelector
```

2启动

`org.apache.rocketmq.namesrv.NamesrvStartup#start`

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {
    ...

    //初始化
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }

    //注册ShutDownHook
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));
    //启动
    controller.start();

    return controller;
}
```

`NamesrvController`生命周期

1. `org.apache.rocketmq.namesrv.NamesrvController#initialize`
2. `org.apache.rocketmq.namesrv.NamesrvController#start`
3. `org.apache.rocketmq.namesrv.NamesrvController#shutdown`

```java
public boolean initialize() {

    //加载KV配置
    this.kvConfigManager.load();
    //创建NettyServer网络处理对象
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

    this.registerProcessor();

    //每10秒扫描broker 移除不活跃的broker
    this.scheduledExecutorService.scheduleAtFixedRate(()->{
         NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }, 5, 10, TimeUnit.SECONDS);

    //每10分钟打印KV配置
    this.scheduledExecutorService.scheduleAtFixedRate(()->{
            NamesrvController.this.kvConfigManager.printAllPeriodically();
    }, 1, 10, TimeUnit.MINUTES);
    
    //
    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        ...
    ｝
    return true;
}

public void start() throws Exception {
    this.remotingServer.start();

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}

public void shutdown() {
    this.remotingServer.shutdown();
    this.remotingExecutor.shutdown();
    this.scheduledExecutorService.shutdown();

    if (this.fileWatchService != null) {
        this.fileWatchService.shutdown();
    }
}
```

### Broker

#### Broker启动流程

`org.apache.rocketmq.broker.BrokerStartup`

```java
public static void main(String[] args) {
    start(createBrokerController(args));
}
```

1创建

`org.apache.rocketmq.broker.BrokerStartup#createBrokerController`

```java
public static BrokerController createBrokerController(String[] args) {

    ...
    final BrokerConfig brokerConfig = new BrokerConfig();
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    final NettyClientConfig nettyClientConfig = new NettyClientConfig();

    nettyClientConfig.setUseTLS(
        Boolean.parseBoolean(System.getProperty(TLS_ENABLE,
                String.valueOf(TlsSystemConfig.tlsMode == TlsMode.ENFORCING))));
    nettyServerConfig.setListenPort(10911);
    final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

    if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
        int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
        messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
    }

    ...

    String namesrvAddr = brokerConfig.getNamesrvAddr();
    if (null != namesrvAddr) {
        String[] addrArray = namesrvAddr.split(";");
        for (String addr : addrArray) {
            RemotingUtil.string2SocketAddress(addr);
        }
    }

    switch (messageStoreConfig.getBrokerRole()) {
        case ASYNC_MASTER:
        case SYNC_MASTER:
            brokerConfig.setBrokerId(MixAll.MASTER_ID);
            break;
        case SLAVE:
            if (brokerConfig.getBrokerId() <= 0) {
                System.out.printf("Slave's brokerId must be > 0");
                System.exit(-3);
            }

            break;
        default:
            break;
    }

    messageStoreConfig.setHaListenPort(nettyServerConfig.getListenPort() + 1);
    ...

    final BrokerController controller = new BrokerController(brokerConfig,
                     nettyServerConfig,nettyClientConfig,messageStoreConfig);
    // remember all configs to prevent discard
    controller.getConfiguration().registerConfig(properties);

    //初始化
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }

    Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        private volatile boolean hasShutdown = false;
        private AtomicInteger shutdownTimes = new AtomicInteger(0);

        @Override
        public void run() {
            synchronized (this) {
                log.info("Shutdown hook was invoked, {}", this.shutdownTimes.incrementAndGet());
                if (!this.hasShutdown) {
                    this.hasShutdown = true;
                    long beginTime = System.currentTimeMillis();
                    controller.shutdown();
                    long consumingTimeTotal = System.currentTimeMillis() - beginTime;
                    log.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
                }
            }
        }
    }, "ShutdownHook"));

    return controller;

}
```

`org.apache.rocketmq.common.BrokerConfig`

`org.apache.rocketmq.remoting.netty.NettyServerConfig`

`org.apache.rocketmq.remoting.netty.NettyClientConfig`

`org.apache.rocketmq.store.config.MessageStoreConfig`



2启动

`org.apache.rocketmq.broker.BrokerController#start`

```java
public static BrokerController start(BrokerController controller) {

	//启动
    controller.start();

    //输出日志
    String tip = "The broker[" + controller.getBrokerConfig().getBrokerName() + ", "
        + controller.getBrokerAddr() + "] boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();

    if (null != controller.getBrokerConfig().getNamesrvAddr()) {
        tip += " and name server is " + controller.getBrokerConfig().getNamesrvAddr();
    }

    log.info(tip);
    System.out.printf("%s%n", tip);
    return controller;

}
```

`BrokerController`生命周期

1. `org.apache.rocketmq.broker.BrokerController#initialize`
2. `org.apache.rocketmq.broker.BrokerController#start`
3. `org.apache.rocketmq.broker.BrokerController#shutdown`





## 路由注册、故障剔除、路由发现

#### 路由元信息

`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager`

```java
public class RouteInfoManager {
    //消息队列路由信息，消息发送时根据路由表进行负载均衡
	private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    //基础信息，Broker基础信息，包含brokerName,所属集群名称，主备broker地址
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    //集群信息，存储集群中所有broker名称
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    //broker状态信息，NameServer每次收到心跳包时会替换该信息
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    //broker上的FilterServer列表
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}
```

`org.apache.rocketmq.common.protocol.route.QueueData`

```java
public class QueueData implements Comparable<QueueData> {
    private String brokerName;
    private int readQueueNums;
    private int writeQueueNums;
    //读写权限 org.apache.rocketmq.common.constant.PermName
    private int perm;
    //topic同步标记 org.apache.rocketmq.common.sysflag.TopicSysFlag
    private int topicSynFlag;
}
```

`org.apache.rocketmq.common.protocol.route.BrokerData`

```java
public class BrokerData implements Comparable<BrokerData> {
    private String cluster;
    private String brokerName;
    //brokerId = 0 为Master， brokerId > 0 为Slave
    private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;
    private final Random random = new Random();
}
```

`org.apache.rocketmq.namesrv.routeinfo.BrokerLiveInfo`

```java
class BrokerLiveInfo {
    private long lastUpdateTimestamp;
    private DataVersion dataVersion;
    private Channel channel;
    private String haServerAddr;
}
```

#### 路由注册

1. Broker发送请求包

```java
//默认，每30秒上报一次
this.scheduledExecutorService.scheduleAtFixedRate(()->{
    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
  }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

public synchronized void registerBrokerAll(final boolean checkOrderConfig, 
                                           boolean oneway, boolean forceRegister) {
    TopicConfigSerializeWrapper topicConfigWrapper = 
        this.getTopicConfigManager().buildTopicConfigSerializeWrapper();

    //不可读或不可写的时候，重新new一遍TopicConfig
    if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission())
        || !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {
        ConcurrentHashMap<String, TopicConfig> topicConfigTable = 
            new ConcurrentHashMap<String, TopicConfig>();
        for (TopicConfig topicConfig : topicConfigWrapper
             .getTopicConfigTable().values()) {
            TopicConfig tmp = 
                new TopicConfig(
                topicConfig.getTopicName(),
                topicConfig.getReadQueueNums(),
                topicConfig.getWriteQueueNums(),
                this.brokerConfig.getBrokerPermission());
            
            topicConfigTable.put(topicConfig.getTopicName(), tmp);
        }
        topicConfigWrapper.setTopicConfigTable(topicConfigTable);
    }

    //是否需要注册
    if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
                                      this.getBrokerAddr(),
                                      this.brokerConfig.getBrokerName(),
                                      this.brokerConfig.getBrokerId(),
                                      this.brokerConfig.getRegisterBrokerTimeoutMills())) {
        doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
    }
}

private void doRegisterBrokerAll(boolean checkOrderConfig, boolean oneway,
                                 TopicConfigSerializeWrapper topicConfigWrapper) {
    //注册broker
    List<RegisterBrokerResult> registerBrokerResultList = 
        this.brokerOuterAPI.registerBrokerAll(
        this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId(),
        this.getHAServerAddr(),
        topicConfigWrapper,
        this.filterServerManager.buildNewFilterServerList(),
        oneway,
        this.brokerConfig.getRegisterBrokerTimeoutMills(),
        this.brokerConfig.isCompressedRegister());

    if (registerBrokerResultList.size() > 0) {
        RegisterBrokerResult registerBrokerResult = registerBrokerResultList.get(0);
        if (registerBrokerResult != null) {
            if (this.updateMasterHAServerAddrPeriodically 
                && registerBrokerResult.getHaServerAddr() != null) {
                this.messageStore.updateHaMasterAddress(
                    registerBrokerResult.getHaServerAddr());
            }

            this.slaveSynchronize.setMasterAddr(registerBrokerResult.getMasterAddr());

            if (checkOrderConfig) {
                this.getTopicConfigManager()
                    .updateOrderTopicConfig(registerBrokerResult.getKvTable());
            }
        }
    }
}

public List<RegisterBrokerResult> registerBrokerAll(......) {

    final List<RegisterBrokerResult> registerBrokerResultList = Lists.newArrayList();
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {

        //封装请求包的header
        final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
        requestHeader.setBrokerAddr(brokerAddr);
        requestHeader.setBrokerId(brokerId);
        requestHeader.setBrokerName(brokerName);
        requestHeader.setClusterName(clusterName);
        requestHeader.setHaServerAddr(haServerAddr);
        requestHeader.setCompressed(compressed);

        RegisterBrokerBody requestBody = new RegisterBrokerBody();
        requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
        requestBody.setFilterServerList(filterServerList);
        final byte[] body = requestBody.encode(compressed);
        final int bodyCrc32 = UtilAll.crc32(body);
        requestHeader.setBodyCrc32(bodyCrc32);
        
        final CountDownLatch countDownLatch = 
            new CountDownLatch(nameServerAddressList.size());
        for (final String namesrvAddr : nameServerAddressList) {
            brokerOuterExecutor.execute(()->{
                  try {
                        //注册
                        RegisterBrokerResult result = registerBroker(
                            namesrvAddr,oneway, timeoutMills,requestHeader,body);
                        if (result != null) {
                            registerBrokerResultList.add(result);
                        }

                        log.info("register broker to name server {} OK", namesrvAddr);
                    } catch (Exception e) {
                        log.warn("registerBroker Exception, {}", namesrvAddr, e);
                    } finally {
                        countDownLatch.countDown();
                    }
            });
        }

        try {
            countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
        }
    }

    return registerBrokerResultList;
}

```

2. NameServer处理请求包

`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#processRequest`

 ```java
case RequestCode.REGISTER_BROKER:
    Version brokerVersion = MQVersion.value2Version(request.getVersion());
    if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
        return this.registerBrokerWithFilterServer(ctx, request);
    } else {
        return this.registerBroker(ctx, request);
    }

public RemotingCommand registerBrokerWithFilterServer(ChannelHandlerContext ctx, RemotingCommand request)throws RemotingCommandException {
    //创建response
    final RemotingCommand response = 
        RemotingCommand.createResponseCommand(RegisterBrokerResponseHeader.class);
    final RegisterBrokerResponseHeader responseHeader = 
        (RegisterBrokerResponseHeader) response.readCustomHeader();
    
    //解码requestHeader
    final RegisterBrokerRequestHeader requestHeader =
        (RegisterBrokerRequestHeader)
        request.decodeCommandCustomHeader(RegisterBrokerRequestHeader.class);

    if (!checksum(ctx, request, requestHeader)) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("crc32 not match");
        return response;
    }

    RegisterBrokerBody registerBrokerBody = new RegisterBrokerBody();

    if (request.getBody() != null) {
        try {
            registerBrokerBody = RegisterBrokerBody.decode(request.getBody(), requestHeader.isCompressed());
        } catch (Exception e) {
            throw new RemotingCommandException("Failed to decode RegisterBrokerBody", e);
        }
    } else {
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion()
          .setCounter(new AtomicLong(0));
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion()
            .setTimestamp(0);
    }

    //修改RouteInfo
    RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
        requestHeader.getClusterName(),
        requestHeader.getBrokerAddr(),
        requestHeader.getBrokerName(),
        requestHeader.getBrokerId(),
        requestHeader.getHaServerAddr(),
        registerBrokerBody.getTopicConfigSerializeWrapper(),
        registerBrokerBody.getFilterServerList(),
        ctx.channel());

    responseHeader.setHaServerAddr(result.getHaServerAddr());
    responseHeader.setMasterAddr(result.getMasterAddr());

    byte[] jsonValue = this.namesrvController.getKvConfigManager().getKVListByNamespace(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG);
    response.setBody(jsonValue);

    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
}
public RegisterBrokerResult registerBroker(......) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            //加写锁
            this.lock.writeLock().lockInterruptibly();
			//判断集群的Broker是否存在
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            //修改broker信息
            boolean registerFirst = false;
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;
                brokerData = new BrokerData(clusterName, brokerName, 
                                            new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);
			
            //masterBroker
            if (null != topicConfigWrapper
                && MixAll.MASTER_ID == brokerId) {
                //修改过brokerTopicConfig或者是第一次注册
                if (this.isBrokerTopicConfigChanged(
                    brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }

            //更新存活Broker信息
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable
                .put(brokerAddr,new BrokerLiveInfo(System.currentTimeMillis(),
                                                   topicConfigWrapper.getDataVersion(),
                                                   channel,haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info("new broker registered, {} HAServer: {}", 
                         brokerAddr, haServerAddr);
            }

            //注册Broker的FilterServe
            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }

            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    //返回主节点信息
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}
 ```

NameServer和Broker保持长连接，

Broker的状态保存在`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#brokerLiveTable`中，

NameServer每收到一个心跳包将更新`topicQueueTable`、`brokerAddrTable`、`clusterAddrTable`、`brokerLiveTable`、`filterServerTable`。

#### 路由剔除

1. 每10秒扫描broker 移除不活跃的broker
2. Broker正常关闭，调用`org.apache.rocketmq.broker.BrokerController#unregisterBrokerAll`

```java
//每10秒扫描broker 移除不活跃的broker
this.scheduledExecutorService.scheduleAtFixedRate(()->{
    NamesrvController.this.routeInfoManager.scanNotActiveBroker();
}, 5, 10, TimeUnit.SECONDS);
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = 
        this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        //超过120秒未更新时移除Broker,关闭Channel
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            it.remove();
            log.warn("The broker channel expired, {} {}ms", 
                     next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
public void onChannelDestroy(String remoteAddr, Channel channel) {
    //找到待删除的Broker的地址
    String brokerAddrFound = null;
    if (channel != null) {
        try {
            try {
                this.lock.readLock().lockInterruptibly();
                Iterator<Entry<String, BrokerLiveInfo>> itBrokerLiveTable =
                    this.brokerLiveTable.entrySet().iterator();
                while (itBrokerLiveTable.hasNext()) {
                    Entry<String, BrokerLiveInfo> entry = itBrokerLiveTable.next();
                    if (entry.getValue().getChannel() == channel) {
                        brokerAddrFound = entry.getKey();
                        break;
                    }
                }
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (Exception e) {
            log.error("onChannelDestroy Exception", e);
        }
    }

    if (null == brokerAddrFound) {
        brokerAddrFound = remoteAddr;
    } else {
        log.info("the broker's channel destroyed, {}, 
                 clean it's data structure at once", brokerAddrFound);
    }

    if (brokerAddrFound != null && brokerAddrFound.length() > 0) {

        try {
            try {
                //申请写锁，修改 brokerLiveTable和 filterServerTable
                this.lock.writeLock().lockInterruptibly();
                this.brokerLiveTable.remove(brokerAddrFound);
                this.filterServerTable.remove(brokerAddrFound);
                
                //更新brokerAddrTable
                String brokerNameFound = null;
                boolean removeBrokerName = false;
                Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                    this.brokerAddrTable.entrySet().iterator();
                while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
                    BrokerData brokerData = itBrokerAddrTable.next().getValue();

                    Iterator<Entry<Long, String>> it = 
                        brokerData.getBrokerAddrs().entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<Long, String> entry = it.next();
                        Long brokerId = entry.getKey();
                        String brokerAddr = entry.getValue();
                        if (brokerAddr.equals(brokerAddrFound)) {
                            brokerNameFound = brokerData.getBrokerName();
                            it.remove();
                            log.info("remove brokerAddr[{}, {}] from brokerAddrTable,"+
                                     "because channel destroyed",brokerId, brokerAddr);
                            break;
                        }
                    }

                    if (brokerData.getBrokerAddrs().isEmpty()) {
                        removeBrokerName = true;
                        itBrokerAddrTable.remove();
                        log.info("remove brokerName[{}] from brokerAddrTable,"+
                                 "because channel destroyed",brokerData.getBrokerName());
                    }
                }
                                 
                //更新clusterAddrTable
                if (brokerNameFound != null && removeBrokerName) {
                    Iterator<Entry<String, Set<String>>> it = 
                        this.clusterAddrTable.entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<String, Set<String>> entry = it.next();
                        String clusterName = entry.getKey();
                        Set<String> brokerNames = entry.getValue();
                        boolean removed = brokerNames.remove(brokerNameFound);
                        if (removed) {
                            log.info("remove brokerName[{}], clusterName[{}]"+ 
                                     "from clusterAddrTable, because channel destroyed",
                                     brokerNameFound, clusterName);

                            if (brokerNames.isEmpty()) {
                                log.info("remove the clusterName[{}] from"+
                                         "clusterAddrTable, because channel destroyed"+
                                         "and no broker in this cluster",clusterName);
                                it.remove();
                            }
                            break;
                        }
                    }
                }

                //删除topicQueueTable
                if (removeBrokerName) {
                    Iterator<Entry<String, List<QueueData>>> itTopicQueueTable =
                        this.topicQueueTable.entrySet().iterator();
                    while (itTopicQueueTable.hasNext()) {
                        Entry<String, List<QueueData>> entry = itTopicQueueTable.next();
                        String topic = entry.getKey();
                        List<QueueData> queueDataList = entry.getValue();

                        Iterator<QueueData> itQueueData = queueDataList.iterator();
                        while (itQueueData.hasNext()) {
                            QueueData queueData = itQueueData.next();
                            if (queueData.getBrokerName().equals(brokerNameFound)) {
                                itQueueData.remove();
                                log.info("remove topic[{} {}], from topicQueueTable,"+
                                         "because channel destroyed",topic, queueData);
                            }
                        }

                        if (queueDataList.isEmpty()) {
                            itTopicQueueTable.remove();
                            log.info("remove topic[{}] all queue, from topicQueueTable,"+
                                     "because channel destroyed",topic);
                        }
                    }
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("onChannelDestroy Exception", e);
        }
    }
}
```



#### 路由发现

`org.apache.rocketmq.common.protocol.route.TopicRouteData`

```java
public class TopicRouteData extends RemotingSerializable {
    //顺序消息配置内容
    private String orderTopicConf;
    //topic队列元数据
    private List<QueueData> queueDatas;
    //topic分布的Broker元数据
    private List<BrokerData> brokerDatas;
    //broker上过滤服务器地址列表
    private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}
```

NameServer不主动推送费客户端，由客户端定时拉取主题最新的路由

```java
case RequestCode.GET_ROUTEINTO_BY_TOPIC:
	return this.getRouteInfoByTopic(ctx, request);
```

```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
                                           RemotingCommand request) 
    throws RemotingCommandException {
    //创建response
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    
    //解析requestHeader
    final GetRouteInfoRequestHeader requestHeader = (GetRouteInfoRequestHeader)
        request.decodeCommandCustomHeader(GetRouteInfoRequestHeader.class);

    //获取topic路由信息
    TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager()
        .pickupTopicRouteData(requestHeader.getTopic());

    if (topicRouteData != null) {
        if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
            //获取KVConfig信息
            String orderTopicConf =
                this.namesrvController.getKvConfigManager()
                .getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                             requestHeader.getTopic());
            
            topicRouteData.setOrderTopicConf(orderTopicConf);
        }

        byte[] content = topicRouteData.encode();
        response.setBody(content);
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }

    response.setCode(ResponseCode.TOPIC_NOT_EXIST);
    response.setRemark("No topic route info in name server for the topic: " +
                       requestHeader.getTopic()
                       + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
    return response;
}

public TopicRouteData pickupTopicRouteData(final String topic) {
    TopicRouteData topicRouteData = new TopicRouteData();
    boolean foundQueueData = false;
    boolean foundBrokerData = false;
    Set<String> brokerNameSet = new HashSet<String>();
    List<BrokerData> brokerDataList = new LinkedList<BrokerData>();
    topicRouteData.setBrokerDatas(brokerDataList);

    HashMap<String, List<String>> filterServerMap = new HashMap<String, List<String>>();
    topicRouteData.setFilterServerTable(filterServerMap);

    try {
        try {
            this.lock.readLock().lockInterruptibly();
            List<QueueData> queueDataList = this.topicQueueTable.get(topic);
            if (queueDataList != null) {
                topicRouteData.setQueueDatas(queueDataList);
                foundQueueData = true;

                Iterator<QueueData> it = queueDataList.iterator();
                while (it.hasNext()) {
                    QueueData qd = it.next();
                    brokerNameSet.add(qd.getBrokerName());
                }

                for (String brokerName : brokerNameSet) {
                    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                    if (null != brokerData) {
                        BrokerData brokerDataClone = 
                            new BrokerData(brokerData.getCluster(),
                                           brokerData.getBrokerName(), 
                                           (HashMap<Long, String>)
                                           brokerData.getBrokerAddrs().clone());
                        brokerDataList.add(brokerDataClone);
                        foundBrokerData = true;
                        for (final String brokerAddr : brokerDataClone.getBrokerAddrs()
                             .values()) {
                            List<String> filterServerList = 
                                this.filterServerTable.get(brokerAddr);
                            
                            filterServerMap.put(brokerAddr, filterServerList);
                        }
                    }
                }
            }
        } finally {
            this.lock.readLock().unlock();
        }
    } catch (Exception e) {
        log.error("pickupTopicRouteData Exception", e);
    }

    log.debug("pickupTopicRouteData {} {}", topic, topicRouteData);

    if (foundBrokerData && foundQueueData) {
        return topicRouteData;
    }

    return null;
}
```

## 消息发送

> 同步：调用消息发送API后，同步等待，直到消息服务器返回发送结果
>
> 异步：调用消息发送API后，立刻返回，消息发送者线程不阻塞。
>
> 单向：调用消息发送API后，直接返回，不管消息是否成功存储在消息服务器上。

### 消息`Message`

`org.apache.rocketmq.common.message.Message`

```java
public class Message implements Serializable {

    //主题
    private String topic;
    //消息Flag org.apache.rocketmq.common.sysflag.MessageSysFlag
    private int flag;
    //扩展属性
    private Map<String, String> properties;
    //消息体
    private byte[] body;
    //事务号
    private String transactionId;
}

//Mesage索引键
public static final String PROPERTY_KEYS = "KEYS";
//用于消息过滤
public static final String PROPERTY_TAGS = "TAGS";
//是否等待消息存储完成后在返回
public static final String PROPERTY_WAIT_STORE_MSG_OK = "WAIT";
//消息延时级别，用于定时消息或消息重试
public static final String PROPERTY_DELAY_TIME_LEVEL = "DELAY";
```

### 生产者启动流程

- `org.apache.rocketmq.client.MQAdmin`
  - `org.apache.rocketmq.client.producer.MQProducer`
    - `org.apache.rocketmq.client.producer.DefaultMQProducer`

`MQAdmin`

创建主题

`org.apache.rocketmq.client.MQAdmin#createTopic`

根据时间戳从队列中查找偏移量

`org.apache.rocketmq.client.MQAdmin#searchOffset`

查找消息队列中最大物理偏移量

`org.apache.rocketmq.client.MQAdmin#maxOffset`

查找消息队列中最小物理偏移量

`org.apache.rocketmq.client.MQAdmin#minOffset`

根据消息偏移量查找消息

`org.apache.rocketmq.client.MQAdmin#viewMessage`

条件查询消息

`org.apache.rocketmq.client.MQAdmin#queryMessage`



`MQProducer`

查找主题下所有消息队列

`org.apache.rocketmq.client.producer.MQProducer#fetchPublishMessageQueues`

同步发送消息/异步发送消息/批量发送消息

`org.apache.rocketmq.client.producer.MQProducer#send`

消息单向发送

`org.apache.rocketmq.client.producer.MQProducer#sendOneway`



`org.apache.rocketmq.client.producer.DefaultMQProducer#start`

```java
public void start() throws MQClientException {
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}

public void start(final boolean startFactory) throws MQClientException {
    //检查是否符合要求
    this.checkConfig();

    if (!this.defaultMQProducer.getProducerGroup()
        .equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
        this.defaultMQProducer.changeInstanceNameToPID();
    }

    //创建MQClientInstance实例
    this.mQClientFactory = MQClientManager.getInstance()
        .getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

    //向MQClientInstance注册
    boolean registerOK = mQClientFactory
        .registerProducer(this.defaultMQProducer.getProducerGroup(), this);
    if (!registerOK) {
        this.serviceState = ServiceState.CREATE_JUST;
        throw new MQClientException(...);
    }

    //存储Toplic信息
    this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(),
                                   new TopicPublishInfo());

    if (startFactory) {
        //启动
        mQClientFactory.start();
    }

    this.serviceState = ServiceState.RUNNING;
}

```

### 消息发送基本流程

`org.apache.rocketmq.client.producer.MQProducer#send`

```java
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(msg);
}

public SendResult send(Message msg) throws 
    MQClientException,RemotingException,MQBrokerException,
InterruptedException {
    return send(
        msg, this.defaultMQProducer.getSendMsgTimeout());
}

public SendResult send(Message msg,long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendDefaultImpl(
        msg, CommunicationMode.SYNC, null, timeout);
}
```



`org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl`

1. 验证状态，验证消息

```java
this.makeSureStateOK();
Validators.checkMessage(msg, this.defaultMQProducer);
```

2. 查找主题路由信息

```java
public class TopicPublishInfo {
    // 是否顺序消息
    private boolean orderTopic = false;
    private boolean haveTopicRouterInfo = false;
    // 消息队列
    private List<MessageQueue> messageQueueList = 
        new ArrayList<MessageQueue>();
    // 没选择一次消息队列，会自增1
    private volatile ThreadLocalIndex sendWhichQueue = 
        new ThreadLocalIndex();
    private TopicRouteData topicRouteData;
}
```

```java
TopicPublishInfo topicPublishInfo = 
    this.tryToFindTopicPublishInfo(msg.getTopic());

private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    TopicPublishInfo topicPublishInfo = 
        this.topicPublishInfoTable.get(topic);
    
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, 
                                               new TopicPublishInfo());
        
        //更新路由
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        //更新路由
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(
            topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}

```

3. 发送消息

```java
int timesTotal = communicationMode == CommunicationMode.SYNC ? 
    1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
int times = 0;
String[] brokersSent = new String[timesTotal];
//循环重试次数
for (; times < timesTotal; times++) {
    String lastBrokerName = null == mq ? null : mq.getBrokerName();
    //选择消息队列
    MessageQueue mqSelected = this.selectOneMessageQueue(
        topicPublishInfo, lastBrokerName);
    if (mqSelected != null) {
        mq = mqSelected;
        brokersSent[times] = mq.getBrokerName();
       
        beginTimestampPrev = System.currentTimeMillis();
        long costTime = beginTimestampPrev - beginTimestampFirst;
        //超时
        if (timeout < costTime) {
            callTimeout = true;
            break;
        }

        //消息发送
        sendResult = this.sendKernelImpl(msg, mq, communicationMode, 
                                         sendCallback, topicPublishInfo, 
                                         timeout - costTime);
        endTimestamp = System.currentTimeMillis();
        this.updateFaultItem(mq.getBrokerName(), 
                             endTimestamp - beginTimestampPrev, false);
    }
}                       
```
- 选择消息队列

  `org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#selectOneMessageQueue`

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    return this.mqFaultStrategy.selectOneMessageQueue(
        tpInfo, lastBrokerName);
}
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    //是否启用故障延迟机制
    if (this.sendLatencyFaultEnable) {
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) %
                    tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                //消息队列是否可用
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName ||
                        mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            final String notBestBroker =
                latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(
                        tpInfo.getSendWhichQueue().getAndIncrement() %
                        writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}

public interface LatencyFaultTolerance<T> {
    // 更新失败条目
    void updateFaultItem(final T name, final long currentLatency, 
                         final long notAvailableDuration);
	// 判断broker是否可用
    boolean isAvailable(final T name);

    // 移除Fault条目
    void remove(final T name);

    // 从规避条目中选择一个
    T pickOneAtLeast();
}
class FaultItem implements Comparable<FaultItem> {
    //brokerName
    private final String name;
    //本次消息发送延迟
    private volatile long currentLatency;
    //故障规避开始时间
    private volatile long startTimestamp;
｝
```

- 消息发送

  `org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendKernelImpl`

```java
//根据broker获取地址
String brokerAddr =
    this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
if (null == brokerAddr) {
    //主动更新
    tryToFindTopicPublishInfo(mq.getTopic());
    brokerAddr =
        this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
}
//设置唯一ID
if (!(msg instanceof MessageBatch)) {
    MessageClientIDSetter.setUniqID(msg);
}

int sysFlag = 0;
boolean msgBodyCompressed = false;
//压缩消息
if (this.tryToCompressMessage(msg)) {
    sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
    msgBodyCompressed = true;
}

final String tranMsg = 
    msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
    sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
}
//执行消息发送前的增强逻辑
if (this.hasSendMessageHook()) {
    context = new SendMessageContext();
    context.setProducer(this);
    context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
    context.setCommunicationMode(communicationMode);
    context.setBornHost(this.defaultMQProducer.getClientIP());
    context.setBrokerAddr(brokerAddr);
    context.setMessage(msg);
    context.setMq(mq);
    String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
    if (isTrans != null && isTrans.equals("true")) {
        context.setMsgType(MessageType.Trans_Msg_Half);
    }

    if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
        context.setMsgType(MessageType.Delay_Msg);
    }
    this.executeSendMessageHookBefore(context);
}

//构建消息发送请求包
SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
//选择发送模型
switch (communicationMode) {
    case ASYNC:
        ...
    case ONEWAY:
    case SYNC:
    default:
｝
/*
发送消息方法
sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(...);
*/

//执行消息发钩子函数
if (this.hasSendMessageHook()) {
    context.setSendResult(sendResult);
    this.executeSendMessageHookAfter(context);
}
```

Broker处理类

`org.apache.rocketmq.broker.processor.SendMessageProcessor`

### 批量消息发送

`org.apache.rocketmq.remoting.protocol.RemotingCommand`

```java
public class RemotingCommand {
    //请求命令编码
	private int code;
    private LanguageCode language = LanguageCode.JAVA;
    //版本号
    private int version = 0;
    //客户端请求号
    private int opaque = requestId.getAndIncrement();
    //标记 倒数第一位表示请求类型,0:请求,1:返回;倒数第二位表示oneway,
    private int flag = 0;
    //描述
    private String remark;
    //扩展属性
    private HashMap<String, String> extFields;
    //每个请求对应的请求头信息
    private transient CommandCustomHeader customHeader;

    private SerializeType serializeTypeCurrentRPC =
        serializeTypeConfigInThisServer;
	//消息体内容
    private transient byte[] body;
    /*
     * 单挑消息发送时，消息体内容将保存在body中。
     * 批量发送时，需要将多条消息的内容存储在body中。
     * |总长度4字节|魔数4字节|bodyCRC4字节|flag4字节|body长度4字节|消息体N字节|
       属性长度2字节|扩展属性N字节|
     */
    
}
```

```java
public SendResult send(
    Collection<Message> msgs) throws MQClientException, RemotingException,
MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(batch(msgs));
}
private MessageBatch batch(Collection<Message> msgs) throws MQClientException {
    MessageBatch msgBatch;
    try {
        msgBatch = MessageBatch.generateFromList(msgs);
        for (Message message : msgBatch) {
            Validators.checkMessage(message, this);
            MessageClientIDSetter.setUniqID(message);
        }
        msgBatch.setBody(msgBatch.encode());
    } catch (Exception e) {
        throw new MQClientException("Failed to initiate the MessageBatch", e);
    }
    return msgBatch;
}

//org.apache.rocketmq.common.message.MessageDecoder#encodeMessages
//org.apache.rocketmq.common.message.MessageDecoder#encodeMessage
```



> 1. 消息生产者启动流程`org.apache.rocketmq.client.impl.factory.MQClientInstance`和`org.apache.rocketmq.client.producer.MQProducer`关系
> 2. 消息队列负载 
> 3. 消息发送异常 重试和规避异常
> 4. 消息批量发送

## 消息存储

> 存储模型
>
> - 持久化
>   - 文件系统
>   - KV存储
>   - 关系型数据库
> - 非持久化
>
> I/O访问性能
>
> `org.apache.rocketmq.store.CommitLog`
>
> 消息存储文件，存储有消息主题的消息
>
> `org.apache.rocketmq.store.ConsumeQueue`
>
> 消息消费队列，消息到达CommitLog后，异步转发到消息队列
>
> `org.apache.rocketmq.store.index.IndexFile`
>
> 消息索引文件，存储key与offset的对应关系
>
> 事务状态服务
>
> 定时消息服务

### 消息存储过程

- `org.apache.rocketmq.store.MessageStore`
  - `org.apache.rocketmq.store.DefaultMessageStore`

```java
public class DefaultMessageStore implements MessageStore {
    //消息存储配置属性
	private final MessageStoreConfig messageStoreConfig;
    //CommitLog文件存储实现类
    private final CommitLog commitLog;
    //消息队列缓存表
    private final ConcurrentMap<String/* topic */, 
    ConcurrentMap<Integer/* queueId */, ConsumeQueue>> consumeQueueTable;
    //消息队列文件ConsumeQueue刷盘线程
    private final FlushConsumeQueueService flushConsumeQueueService;
	//清除CommitLog服务
    private final CleanCommitLogService cleanCommitLogService;
	//清除ConsumeQueue服务
    private final CleanConsumeQueueService cleanConsumeQueueService;
	//索引文件实现类
    private final IndexService indexService;
	//MappedFile分配服务
    private final AllocateMappedFileService allocateMappedFileService;
	//CommitLog消息分发，根据CommitLog文件构建ConsumeQueue/IndexFile文件
    private final ReputMessageService reputMessageService;
	//存储HA机制
    private final HAService haService;

    private final ScheduleMessageService scheduleMessageService;

    private final StoreStatsService storeStatsService;
	//消息堆内缓存
    private final TransientStorePool transientStorePool;
    //消息拉取长轮训模式消息到达监听器
    private final MessageArrivingListener messageArrivingListener;
    //Borker配置属性
    private final BrokerConfig brokerConfig;
    //文件刷盘检测点
    private StoreCheckpoint storeCheckpoint;
    //CommitLog文件转发请求
    private final LinkedList<CommitLogDispatcher> dispatcherList;
}
```

`org.apache.rocketmq.store.MessageStore#putMessage`

- `org.apache.rocketmq.store.DefaultMessageStore#putMessage`

1. 判断是否可以写入

```java
//是否停止
if (this.shutdown) {
    log.warn("message store has shutdown, so putMessage is forbidden");
    return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
}

//是否为slave
if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
    long value = this.printTimes.getAndIncrement();
    if ((value % 50000) == 0) {
        log.warn("message store is slave mode, so putMessage is forbidden ");
    }

    return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
}

//是否可以写入
if (!this.runningFlags.isWriteable()) {
    long value = this.printTimes.getAndIncrement();
    if ((value % 50000) == 0) {
        log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());
    }

    return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
} else {
    this.printTimes.set(0);
}
//topic长度
if (msg.getTopic().length() > Byte.MAX_VALUE) {
    log.warn("putMessage message topic length too long " + msg.getTopic().length());
    return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
}

//额外字段长度
if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {
    log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
    return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
}
//OS pagecache 是否繁忙
if (this.isOSPageCacheBusy()) {
    return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
}
```

2. 写入`CommitLog`

```java
long beginTime = this.getSystemClock().now();
PutMessageResult result = this.commitLog.putMessage(msg);

long eclipseTime = this.getSystemClock().now() - beginTime;
if (eclipseTime > 500) {
    log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
}
this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);

if (null == result || !result.isOk()) {
    this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
}
```

写入流程

`org.apache.rocketmq.store.CommitLog#putMessage`

```java
//消息延时级别大于0 修改topic和queueID ,原生的topic和queueID存入properties
if (msg.getDelayTimeLevel() > 0) {
    if (msg.getDelayTimeLevel() >
        this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
        msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService()
                              .getMaxDelayLevel());
    }

    topic = ScheduleMessageService.SCHEDULE_TOPIC;
    queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

    // Backup real topic, queueId
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC,
                                msg.getTopic());
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID,
                                String.valueOf(msg.getQueueId()));
    msg.setPropertiesString(MessageDecoder.messageProperties2String(
        msg.getProperties()));

    msg.setTopic(topic);
    msg.setQueueId(queueId);
}

//获取可以写入的commitlog文件 ${ROCKETMQ_HOME}/store/commitlog
MappedFile unlockMappedFile = null;
MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();


msg.setStoreTimestamp(beginLockTimestamp);

if (null == mappedFile || mappedFile.isFull()) {
    //未获取到，创建org.apache.rocketmq.store.MappedFile
    mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
}
if (null == mappedFile) {
    //创建失败
    log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " 
              + msg.getBornHostString());
    beginTimeInLock = 0;
    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
}
//消息写入内存
result = mappedFile.appendMessage(msg, this.appendMessageCallback);
// Statistics
storeStatsService.getSinglePutMessageTopicTimesTotal(
    msg.getTopic()).incrementAndGet();
storeStatsService.getSinglePutMessageTopicSizeTotal(
    topic).addAndGet(result.getWroteBytes());

//内存数据持久化到硬盘
handleDiskFlush(result, putMessageResult, msg);
//主从复制
handleHA(result, putMessageResult, msg);
```

写消息

`org.apache.rocketmq.store.MappedFile#appendMessagesInner`

```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    assert messageExt != null;
    assert cb != null;

    int currentPos = this.wrotePosition.get();

    if (currentPos < this.fileSize) {
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() :
        this.mappedByteBuffer.slice();
        byteBuffer.position(currentPos);
        AppendMessageResult result = null;
        if (messageExt instanceof MessageExtBrokerInner) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, 
                                 this.fileSize - currentPos, 
                                 (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer,
                                 this.fileSize - currentPos, 
                                 (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}",
              currentPos, this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```

### 存储文件组织与内存映射

`org.apache.rocketmq.store.MappedFile`

```java
public class MappedFile extends ReferenceResource {
    // 操作系统每页大小
    public static final int OS_PAGE_SIZE = 1024 * 4;
	// 当前JVM实例中MappedFile虚拟内存
    private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);
	// 当前JVM实例中MappedFile对象个数
    private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);
    //当前该文件的写指针
    protected final AtomicInteger wrotePosition = new AtomicInteger(0);
    //当前文件的提交指针
    protected final AtomicInteger committedPosition = new AtomicInteger(0);
    //该指针之前的数据持久化到磁盘
    private final AtomicInteger flushedPosition = new AtomicInteger(0);
   
    protected int fileSize;
    
    protected FileChannel fileChannel;
    //堆内存 数据首先存储在这，然后提交到MappedFile对应的内存映射文件
    protected ByteBuffer writeBuffer = null;
    //堆内存池
    protected TransientStorePool transientStorePool = null;
    private String fileName;
    //文件初始偏移量
    private long fileFromOffset;
    private File file;
    //物理文件对应的内存映射buffer
    private MappedByteBuffer mappedByteBuffer;
    //文件最后一次内容写入时间
    private volatile long storeTimestamp = 0;
    private boolean firstCreateInQueue = false;
}
```

初始化`org.apache.rocketmq.store.MappedFile#init`

内存映射文件提交`org.apache.rocketmq.store.MappedFile#commit`

将内存中的数据刷写到磁盘`org.apache.rocketmq.store.MappedFile#flush`

获取当前文件最大的可读指针`org.apache.rocketmq.store.MappedFile#getReadPosition`

查找pos到当前最大可读之间的数据`org.apache.rocketmq.store.MappedFile#selectMappedBuffer`

文件销毁`org.apache.rocketmq.store.MappedFile#destroy`

`org.apache.rocketmq.store.MappedFileQueue`

```java
public class MappedFileQueue {
    //存储目录
    private final String storePath;
	//单个文件的存储大小
    private final int mappedFileSize;
	//MappedFile文件集合
    private final CopyOnWriteArrayList<MappedFile> mappedFiles = new CopyOnWriteArrayList<MappedFile>();
	//创建服务类
    private final AllocateMappedFileService allocateMappedFileService;
	//当前刷盘指针
    private long flushedWhere = 0;
    //当前数据提交指针
    private long committedWhere = 0;

    private volatile long storeTimestamp = 0;
}
```

根据消息存储时间戳查询`org.apache.rocketmq.store.MappedFileQueue#getMappedFileByTime`

根据消息偏移量offset查找MappedFile`org.apache.rocketmq.store.MappedFileQueue#findMappedFileByOffset`

获取存储文件最小偏移量`org.apache.rocketmq.store.MappedFileQueue#getMinOffset`

获取存储文件最大偏移量`org.apache.rocketmq.store.MappedFileQueue#getMaxOffset`

返回存储文件当前的写指针`org.apache.rocketmq.store.MappedFileQueue#getMaxWrotePosition`

### 实时更新消息消费队列与索引文件

`org.apache.rocketmq.store.DefaultMessageStore#start`

```java
if (this.getMessageStoreConfig().isDuplicationEnable()) {
    this.reputMessageService.setReputFromOffset(this.commitLog.getConfirmOffset());
} else {
    this.reputMessageService.setReputFromOffset(this.commitLog.getMaxOffset());
}
this.reputMessageService.start();
```

`org.apache.rocketmq.store.DefaultMessageStore.ReputMessageService`

```java
public void run() {
    DefaultMessageStore.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            //每执行一次任务休息1ms，继续尝试推送
            Thread.sleep(1);
            this.doReput();
        } catch (Exception e) {
            DefaultMessageStore.log.warn(this.getServiceName() 
                                         + " service has exception. ", e);
        }
    }

    DefaultMessageStore.log.info(this.getServiceName() + " service end");
}

// 1 返回reputFromOffset偏移量
SelectMappedBufferResult result = 
    DefaultMessageStore.this.commitLog.getData(reputFromOffset);
// 2 创建Dispatch-Request对象 
DispatchRequest dispatchRequest = 
  DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(
    result.getByteBuffer(), false, false);
DefaultMessageStore.this.doDispatch(dispatchRequest);
/*
 org.apache.rocketmq.store.CommitLogDispatcher
 构建消息消费队列
 -org.apache.rocketmq.store.DefaultMessageStore.CommitLogDispatcherBuildConsumeQueue
 构建索引文件
 -org.apache.rocketmq.store.DefaultMessageStore.CommitLogDispatcherBuildIndex
*/
 

public void putMessagePositionInfo(DispatchRequest dispatchRequest) {
    //根据消息主题与队列ID获取对应的ConsumeQueue
    ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(),
                                            dispatchRequest.getQueueId());
    //将消息信息写入到ByteBuffer中
    cq.putMessagePositionInfoWrapper(dispatchRequest);
}

//根据消息更新索引文件
public void buildIndex(DispatchRequest req) {
    //获取或创建IndexFile
    IndexFile indexFile = retryGetAndCreateIndexFile();
    if (indexFile != null) {
        long endPhyOffset = indexFile.getEndPhyOffset();
        DispatchRequest msg = req;
        String topic = msg.getTopic();
        String keys = msg.getKeys();
        if (msg.getCommitLogOffset() < endPhyOffset) {
            return;
        }

        final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
        switch (tranType) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE:
            case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                break;
            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                return;
        }

        //消息的唯一键不为空，则添加到Hash索引中
        if (req.getUniqKey() != null) {
            indexFile = putKey(indexFile, msg, buildKey(topic, req.getUniqKey()));
            if (indexFile == null) {
                log.error("putKey error commitlog {} uniqkey {}",
                          req.getCommitLogOffset(), req.getUniqKey());
                return;
            }
        }

        //构建索引键
        if (keys != null && keys.length() > 0) {
            String[] keyset = keys.split(MessageConst.KEY_SEPARATOR);
            for (int i = 0; i < keyset.length; i++) {
                String key = keyset[i];
                if (key.length() > 0) {
                    indexFile = putKey(indexFile, msg, buildKey(topic, key));
                    if (indexFile == null) {
                        log.error("putKey error commitlog {} uniqkey {}",
                                  req.getCommitLogOffset(), req.getUniqKey());
                        return;
                    }
                }
            }
        }
    } else {
        log.error("build index error, stop building index");
    }
}
```

### 消息队列和索引文件恢复

`org.apache.rocketmq.store.DefaultMessageStore#load`

```java
public boolean load() {
    boolean result = true;

    try {
        //判断上一次退出是否正常
        boolean lastExitOK = !this.isTempFileExist();
        log.info("last shutdown {}", lastExitOK ? "normally" : "abnormally");

        if (null != scheduleMessageService) {
            //加载延时队列
            result = result && this.scheduleMessageService.load();
        }

        // load Commit Log
        result = result && this.commitLog.load();

        // load Consume Queue
        result = result && this.loadConsumeQueue();

        if (result) {
            //加载存储检测点
            this.storeCheckpoint = 
                new StoreCheckpoint(StorePathConfigHelper.getStoreCheckpoint(
                                    this.messageStoreConfig.getStorePathRootDir()));

            //加载文件索引
            this.indexService.load(lastExitOK);

            //根据Broker是否正常停止执行不同的恢复策略
            this.recover(lastExitOK);

            log.info("load over, and the max phy offset = {}",
                     this.getMaxPhyOffset());
        }
    } catch (Exception e) {
        log.error("load exception", e);
        result = false;
    }

    if (!result) {
        this.allocateMappedFileService.shutdown();
    }

    return result;
}
```

### 文件刷盘

> 消息存储时先将消息追加到内存，再根据配置的刷盘策略在不同时间进行刷写磁盘
>
> 同步刷盘：消息追加到内存后，同步调用`java.nio.MappedByteBuffer#force`
>
> 异步刷盘：消息在追加到内存后立刻返回给消息发送端。使用一个单独线程按照某一频率执行刷盘操作

`org.apache.rocketmq.store.CommitLog#handleDiskFlush`

```java
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    // Synchronization flush
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig()
        .getFlushDiskType()) {
        final GroupCommitService service = 
            (GroupCommitService) this.flushCommitLogService;
        if (messageExt.isWaitStoreMsgOK()) {
            //构建同步任务
            GroupCommitRequest request = 
                new GroupCommitRequest(result.getWroteOffset() 
                                       + result.getWroteBytes());
            //提交同步任务
            service.putRequest(request);
            
            //等待完成
            boolean flushOK = 
                request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig()
                                     .getSyncFlushTimeout());
            if (!flushOK) {
                log.error("do groupcommit, wait for flush failed, topic: " 
                          + messageExt.getTopic() + " tags: " + messageExt.getTags()
                          + " client address: " + messageExt.getBornHostString());
                putMessageResult.setPutMessageStatus(
                    PutMessageStatus.FLUSH_DISK_TIMEOUT);
            }
        } else {
            service.wakeup();
        }
    } else {
        // Asynchronous flush
        if (!this.defaultMessageStore.getMessageStoreConfig()
            .isTransientStorePoolEnable()) {
            
            flushCommitLogService.wakeup();
        } else {
            commitLogService.wakeup();
        }
    }
}
```

## 消息消费

- `org.apache.rocketmq.client.consumer.MQConsumer`
  - `org.apache.rocketmq.client.consumer.MQPullConsumer`
    - `org.apache.rocketmq.client.consumer.DefaultMQPullConsumer`
  - `org.apache.rocketmq.client.consumer.MQPushConsumer`
    - `org.apache.rocketmq.client.consumer.DefaultMQPushConsumer`

```java
public interface MQConsumer extends MQAdmin {
    /**
     * 发送消息ACK确认
     * @param msg 消息
     * @param delayLevel 消息延时级别
     * @param brokerName 消息服务器名称
     */
    void sendMessageBack(final MessageExt msg, 
                         final int delayLevel, final String brokerName)
        throws RemotingException, MQBrokerException, InterruptedException,
               MQClientException;
    
    /**
     * 获取消费者对主题topic分配了哪些消息队列
     * @param topic 主题名称
     */
    Set<MessageQueue> fetchSubscribeMessageQueues(final String topic) 
        throws MQClientException;
}
```

`org.apache.rocketmq.client.consumer.MQPushConsumer#registerMessageListener`

消息事件监听器

- 并发`org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently`
- 顺序`org.apache.rocketmq.client.consumer.listener.MessageListenerOrderly`

`org.apache.rocketmq.client.consumer.MQPushConsumer#subscribe`

订阅消息

- 基于消息过滤表达式过滤`tag`或`SQL92表达式`
- `org.apache.rocketmq.client.consumer.MessageSelector`

`org.apache.rocketmq.client.consumer.MQPushConsumer#unsubscribe`

取消订阅

`org.apache.rocketmq.client.consumer.DefaultMQPushConsumer`

```java
public class DefaultMQPushConsumer extends ClientConfig implements MQPushConsumer {
    protected final transient DefaultMQPushConsumerImpl defaultMQPushConsumerImpl;
    //消费者所属组
    private String consumerGroup;
    //消息消费模式
    private MessageModel messageModel = MessageModel.CLUSTERING;
    //根据消息进度从消费服务器拉取不到消息时重新计算消费策略
    private ConsumeFromWhere consumeFromWhere = 
        ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET;
    
    private String consumeTimestamp = UtilAll.timeMillisToHumanString3(
        System.currentTimeMillis() - (1000 * 60 * 30));
    //集群模式下消息队列负载均衡
    private AllocateMessageQueueStrategy allocateMessageQueueStrategy;
    //订阅信息
    private Map<String /* topic */, String /* sub expression */> subscription = 
        new HashMap<String, String>();
    //消息业务监听器
    private MessageListener messageListener;
    //消息消费进度存储器
    private OffsetStore offsetStore;
    //消费者最小线程数
    private int consumeThreadMin = 20;
    //消费者最大线程数
    private int consumeThreadMax = 64;
    private long adjustThreadPoolNumsThreshold = 100000;
    //并发消息消费时处理队列最大跨度
    private int consumeConcurrentlyMaxSpan = 2000;
    //每1000次流控后打印流控值日
    private int pullThresholdForQueue = 1000;
    private int pullThresholdSizeForQueue = 100;
    private int pullThresholdForTopic = -1;
    private int pullThresholdSizeForTopic = -1;
    //推模式下拉取任务时间间隔
    private long pullInterval = 0;
    //消息并发消费时一次消费消息条数
    private int consumeMessageBatchMaxSize = 1;
    //每次消息拉取所拉取的条数
    private int pullBatchSize = 32;
	//是否每次拉取消息都更新订阅信息
    private boolean postSubscriptionWhenPull = false;

    private boolean unitMode = false;
    //最大消费重试次数
    private int maxReconsumeTimes = -1;
    //延迟将该队列的消息提交到消费者线程的等待
    private long suspendCurrentQueueTimeMillis = 1000;
    //消息消费超时时间 （分钟）
    private long consumeTimeout = 15;
    private TraceDispatcher traceDispatcher = null;
}
```

### 消费者启动流程

`org.apache.rocketmq.client.consumer.DefaultMQPushConsumer#start`

```java
public void start() throws MQClientException {
    this.defaultMQPushConsumerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
public synchronized void start() throws MQClientException {
    //检查配置
    this.checkConfig();
    
    //1. 构建主题订阅信息
    this.copySubscription();
    
    //2. 初始化MQClientInstance 设置RebalanceImpl
    this.mQClientFactory = MQClientManager.getInstance()
        .getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
    this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer
                                        .getConsumerGroup());
    this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer
                                       .getMessageModel());
    this.rebalanceImpl.setAllocateMessageQueueStrategy(
        this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
    this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

    //3. 初始化消息进度
    if (this.defaultMQPushConsumer.getOffsetStore() != null) {
        this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
    } else {
        switch (this.defaultMQPushConsumer.getMessageModel()) {
            case BROADCASTING:
                //广播，消息存储在消费端
                this.offsetStore = new LocalFileOffsetStore(
                    this.mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup());
                break;
            case CLUSTERING:
                //集群，消息存储在broker
                this.offsetStore = new RemoteBrokerOffsetStore(
                    this.mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup());
                break;
            default:
                break;
        }
        this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
    }
    this.offsetStore.load();
    
	//4. 创建消费端消费线程服务
    if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
        this.consumeOrderly = true;
        this.consumeMessageService =
            new ConsumeMessageOrderlyService(this,
                     (MessageListenerOrderly) this.getMessageListenerInner());
    } 
    else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
        this.consumeOrderly = false;
        this.consumeMessageService = new ConsumeMessageConcurrentlyService(this,
(MessageListenerConcurrently) this.getMessageListenerInner());
    }

    this.consumeMessageService.start();
    
    //5. 注册消费者并启动MQClientInstance
    boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer
                                       .getConsumerGroup(), this);
    if (!registerOK) {
        this.serviceState = ServiceState.CREATE_JUST;
        this.consumeMessageService.shutdown();
        
    }
    mQClientFactory.start();
}
```

### 消息拉取

`org.apache.rocketmq.client.impl.consumer.PullMessageService`

```java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            //获取消息拉取任务
            PullRequest pullRequest = this.pullRequestQueue.take();
            //进行消息拉取
            this.pullMessage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}
private void pullMessage(final PullRequest pullRequest) {
    final MQConsumerInner consumer = 
        this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        impl.pullMessage(pullRequest);
    } else {
        log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
    }
}
```

`org.apache.rocketmq.client.impl.consumer.PullRequest`

```java
public class PullRequest {
    //消费者组
    private String consumerGroup;
    //待拉取消费队列
    private MessageQueue messageQueue;
    //消息处理队列
    private ProcessQueue processQueue;
    //待拉取的MessageQueue偏移量
    private long nextOffset;
    //是否被锁定
    private boolean lockedFirst = false;
}

public class ProcessQueue {
    // 消息存储容器 key为消息在ConsumeQueue中的偏移量 MessageExt为消息实体
	private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<>();
    // 消息临时存储容器 用于处理顺序消息
    private final TreeMap<Long, MessageExt> consumingMsgOrderlyTreeMap = new TreeMap<>();
    // 读写锁
    private final ReadWriteLock lockTreeMap = new ReentrantReadWriteLock();
    // ProcessQueue中总消息数
    private final AtomicLong msgCount = new AtomicLong();
    // ProcessQueue中包含的最大队列偏移量
    private volatile long queueOffsetMax = 0L;
    //是否被丢弃
    private volatile boolean dropped = false;
    //上一次开始消息拉取时间
    private volatile long lastPullTimestamp = System.currentTimeMillis();
    //上一次消息消费时间
    private volatile long lastConsumeTimestamp = System.currentTimeMillis();
｝
```

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#isLockExpired`

判断锁是否过期

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#isPullExpired`

判断`PullMessageService`是否空闲

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#cleanExpiredMsg`

移除消费超时消息

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#putMessage`

添加消息

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#getMaxSpan`

获取当前消息最大间隔

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#removeMessage`

移除消息

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#rollback`

消息重新放入到msgTreeMap

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#commit`

消息清除

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#makeMessageToCosumeAgain`

重新消费该批消息

`org.apache.rocketmq.client.impl.consumer.ProcessQueue#takeMessags`

取出`batchSize`条消息



消息拉取入口`org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage`

```java
final ProcessQueue processQueue = pullRequest.getProcessQueue();
//判断processQueue
if (processQueue.isDropped()) {
    log.info("the pull request[{}] is dropped.", pullRequest.toString());
    return;
}

pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());

try {
    //判断服务状态
    this.makeSureStateOK();
} catch (MQClientException e) {
    log.warn("pullMessage exception, consumer state not ok", e);
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    return;
}

//是否暂停
if (this.isPause()) {
    log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
    return;
}

//消息拉取流控 消费数量 消息大小 消费间隔
long cachedMessageCount = processQueue.getMsgCount().get();
long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);

if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
    if ((queueFlowControlTimes++ % 1000) == 0) {
        ...
    }
    return;
}

if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
    if ((queueFlowControlTimes++ % 1000) == 0) {
        ...
    }
    return;
}

if (!this.consumeOrderly) {
    if (processQueue.getMaxSpan() >
        this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
        this.executePullRequestLater(
            pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
            ...
        }
        return;
    }
}

//拉取主题订阅信息
final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
if (null == subscriptionData) {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    log.warn("find the consumer's subscription failed, {}", pullRequest);
    return;
}

//构建消息拉取系统标记
boolean commitOffsetEnable = false;
long commitOffsetValue = 0L;
if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
    commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
    if (commitOffsetValue > 0) {
        commitOffsetEnable = true;
    }
}

String subExpression = null;
boolean classFilter = false;
SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
if (sd != null) {
    if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
        subExpression = sd.getSubString();
    }

    classFilter = sd.isClassFilterMode();
}

int sysFlag = PullSysFlag.buildSysFlag(
    commitOffsetEnable, // commitOffset
    true, // suspend
    subExpression != null, // subscription
    classFilter // class filter
);


//调用 与服务端交互
try {
    this.pullAPIWrapper.pullKernelImpl(
        pullRequest.getMessageQueue(),
        subExpression,
        subscriptionData.getExpressionType(),
        subscriptionData.getSubVersion(),
        pullRequest.getNextOffset(),
        this.defaultMQPushConsumer.getPullBatchSize(),
        sysFlag,
        commitOffsetValue,
        BROKER_SUSPEND_MAX_TIME_MILLIS,
        CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
        CommunicationMode.ASYNC,
        pullCallback
    );
} catch (Exception e) {
    log.error("pullKernelImpl exception", e);
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
}
```

`org.apache.rocketmq.common.sysflag.PullSysFlag`

```java
//从内存中读取的消费进度大于0，则设置该标记位
private final static int FLAG_COMMIT_OFFSET = 0x1 << 0;
//消息拉取时支持挂起
private final static int FLAG_SUSPEND = 0x1 << 1;
//消息过滤机制为表达式
private final static int FLAG_SUBSCRIPTION = 0x1 << 2;
//消息锅炉机制为类过滤模式
private final static int FLAG_CLASS_FILTER = 0x1 << 3;
```



`org.apache.rocketmq.client.impl.consumer.PullAPIWrapper#pullKernelImpl`

```java
final MessageQueue mq, //消息消费队列
final String subExpression,//消息过滤表达式
final String expressionType,//消息表达式类型
final long subVersion,//
final long offset,//消息拉取偏移量
final int maxNums,//本次拉取最大消息条数
final int sysFlag,//拉取系统标记
final long commitOffset,//当前MessageQueeu消费进度
final long brokerSuspendMaxTimeMillis,//消息拉取过程中允许Broker挂起时间，默认15s
final long timeoutMillis,//消息拉取超时时间
final CommunicationMode communicationMode,//消息拉取模式 默认为异步拉取
final PullCallback pullCallback// 从broker拉取到消息后的回调方法
```

```java
//获取broker地址
FindBrokerResult findBrokerResult =
    this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
    this.recalculatePullFromWhichNode(mq), false);
if (null == findBrokerResult) {
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(mq.getTopic());
    findBrokerResult =
        this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
        this.recalculatePullFromWhichNode(mq), false);
}

//过滤消息
String brokerAddr = findBrokerResult.getBrokerAddr();
if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
    brokerAddr = computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
}
//异步拉取消息
PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
    brokerAddr,
    requestHeader,
    timeoutMillis,
    communicationMode,
    pullCallback);
```

`Broker`端组装消息

`org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest(io.netty.channel.Channel, org.apache.rocketmq.remoting.protocol.RemotingCommand, boolean)`

### 消息队列负载与重新分布机制

`org.apache.rocketmq.client.impl.consumer.RebalanceService`

```java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        this.waitForRunning(waitInterval);
        this.mqClientFactory.doRebalance();
    }

    log.info(this.getServiceName() + " service end");
}

public void doRebalance() {
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            try {
                impl.doRebalance();
            } catch (Throwable e) {
                log.error("doRebalance exception", e);
            }
        }
    }
}
```

`org.apache.rocketmq.client.impl.consumer.RebalanceImpl#doRebalance`

```java
public void doRebalance(final boolean isOrder) {
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
            final String topic = entry.getKey();
            try {
                //针对单个主题对消息队列重新负载
                this.rebalanceByTopic(topic, isOrder);
            } catch (Throwable e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("rebalanceByTopic Exception", e);
                }
            }
        }
    }

    this.truncateMessageQueueNotMyTopic();
}
```

`org.apache.rocketmq.client.consumer.AllocateMessageQueueStrategy`

```java
public interface AllocateMessageQueueStrategy {

    /**
     * Allocating by consumer id
     *
     * @param consumerGroup current consumer group
     * @param currentCID current consumer id
     * @param mqAll message queue set in current topic
     * @param cidAll consumer set in current consumer group
     * @return The allocate result of given strategy
     */
    List<MessageQueue> allocate(
        final String consumerGroup,
        final String currentCID,
        final List<MessageQueue> mqAll,
        final List<String> cidAll
    );

    /**
     * Algorithm name
     *
     * @return The strategy name
     */
    String getName();
}
```

