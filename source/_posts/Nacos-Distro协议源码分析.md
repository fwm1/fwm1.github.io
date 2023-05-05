---
title: Nacos Distro协议源码分析
date: 2022-05-22 15:38:57
tags: [Distro, Nacos]
categories:
---

### 简介

Distro是Nacos内部使用的一种协议，用于实现分布式环境下的数据一致性。本文尝试从源码分析Distro协议的实现细节，希望从中学习到实现思路和良好经验。

### 协议入口DistroProtocol

```java
@Component
public class DistroProtocol {
    
    //节点管理器
    private final ServerMemberManager memberManager;
    //Distro相关组件Holder
    private final DistroComponentHolder distroComponentHolder;
    //Distro任务执行引擎Holder
    private final DistroTaskEngineHolder distroTaskEngineHolder;
    
    private volatile boolean isInitialized = false;
    
    public DistroProtocol(ServerMemberManager memberManager, DistroComponentHolder distroComponentHolder,
            DistroTaskEngineHolder distroTaskEngineHolder) {
        this.memberManager = memberManager;
        this.distroComponentHolder = distroComponentHolder;
        this.distroTaskEngineHolder = distroTaskEngineHolder;
        //初始化时开启任务
        startDistroTask();
    }
    
    private void startDistroTask() {
        if (EnvUtil.getStandaloneMode()) {
            isInitialized = true;
            return;
        }
        //1.开启verify任务
        startVerifyTask();
		//2.开始load任务
        startLoadTask();
    }
    
    private void startLoadTask() {
        DistroCallback loadCallback = new DistroCallback() {
            @Override
            public void onSuccess() {
                isInitialized = true;
            }
            
            @Override
            public void onFailed(Throwable throwable) {
                isInitialized = false;
            }
        };
        GlobalExecutor.submitLoadDataTask(
                new DistroLoadDataTask(memberManager, distroComponentHolder, DistroConfig.getInstance(), loadCallback));
    }
    
    private void startVerifyTask() {
        GlobalExecutor.schedulePartitionDataTimedSync(new DistroVerifyTimedTask(memberManager, distroComponentHolder, distroTaskEngineHolder.getExecuteWorkersManager()),  DistroConfig.getInstance().getVerifyIntervalMillis());
    }
    
    public boolean isInitialized() {
        return isInitialized;
    }
    
    /**
     * Start to sync by configured delay.
     *
     * @param distroKey distro key of sync data
     * @param action    the action of data operation
     */
    public void sync(DistroKey distroKey, DataOperation action) {
        sync(distroKey, action, DistroConfig.getInstance().getSyncDelayMillis());
    }
    
    /**
     * Start to sync data to all remote server.
     *
     * @param distroKey distro key of sync data
     * @param action    the action of data operation
     * @param delay     delay time for sync
     */
    public void sync(DistroKey distroKey, DataOperation action, long delay) {
        for (Member each : memberManager.allMembersWithoutSelf()) {
            syncToTarget(distroKey, action, each.getAddress(), delay);
        }
    }
    
    /**
     * Start to sync to target server.
     *
     * @param distroKey    distro key of sync data
     * @param action       the action of data operation
     * @param targetServer target server
     * @param delay        delay time for sync
     */
    public void syncToTarget(DistroKey distroKey, DataOperation action, String targetServer, long delay) {
        DistroKey distroKeyWithTarget = new DistroKey(distroKey.getResourceKey(), distroKey.getResourceType(),
                targetServer);
        DistroDelayTask distroDelayTask = new DistroDelayTask(distroKeyWithTarget, action, delay);
        distroTaskEngineHolder.getDelayTaskExecuteEngine().addTask(distroKeyWithTarget, distroDelayTask);
        if (Loggers.DISTRO.isDebugEnabled()) {
            Loggers.DISTRO.debug("[DISTRO-SCHEDULE] {} to {}", distroKey, targetServer);
        }
    }
    
    /**
     * Query data from specified server.
     *
     * @param distroKey data key
     * @return data
     */
    public DistroData queryFromRemote(DistroKey distroKey) {
        if (null == distroKey.getTargetServer()) {
            Loggers.DISTRO.warn("[DISTRO] Can't query data from empty server");
            return null;
        }
        String resourceType = distroKey.getResourceType();
        DistroTransportAgent transportAgent = distroComponentHolder.findTransportAgent(resourceType);
        if (null == transportAgent) {
            Loggers.DISTRO.warn("[DISTRO] Can't find transport agent for key {}", resourceType);
            return null;
        }
        return transportAgent.getData(distroKey, distroKey.getTargetServer());
    }
    
    /**
     * Receive synced distro data, find processor to process.
     *
     * @param distroData Received data
     * @return true if handle receive data successfully, otherwise false
     */
    public boolean onReceive(DistroData distroData) {
        Loggers.DISTRO.info("[DISTRO] Receive distro data type: {}, key: {}", distroData.getType(),
                distroData.getDistroKey());
        String resourceType = distroData.getDistroKey().getResourceType();
        DistroDataProcessor dataProcessor = distroComponentHolder.findDataProcessor(resourceType);
        if (null == dataProcessor) {
            Loggers.DISTRO.warn("[DISTRO] Can't find data process for received data {}", resourceType);
            return false;
        }
        return dataProcessor.processData(distroData);
    }
    
    /**
     * Receive verify data, find processor to process.
     *
     * @param distroData    verify data
     * @param sourceAddress source server address, might be get data from source server
     * @return true if verify data successfully, otherwise false
     */
    public boolean onVerify(DistroData distroData, String sourceAddress) {
        if (Loggers.DISTRO.isDebugEnabled()) {
            Loggers.DISTRO.debug("[DISTRO] Receive verify data type: {}, key: {}", distroData.getType(),
                    distroData.getDistroKey());
        }
        String resourceType = distroData.getDistroKey().getResourceType();
        DistroDataProcessor dataProcessor = distroComponentHolder.findDataProcessor(resourceType);
        if (null == dataProcessor) {
            Loggers.DISTRO.warn("[DISTRO] Can't find verify data process for received data {}", resourceType);
            return false;
        }
        return dataProcessor.processVerifyData(distroData, sourceAddress);
    }
    
    /**
     * Query data of input distro key.
     *
     * @param distroKey key of data
     * @return data
     */
    public DistroData onQuery(DistroKey distroKey) {
        String resourceType = distroKey.getResourceType();
        DistroDataStorage distroDataStorage = distroComponentHolder.findDataStorage(resourceType);
        if (null == distroDataStorage) {
            Loggers.DISTRO.warn("[DISTRO] Can't find data storage for received key {}", resourceType);
            return new DistroData(distroKey, new byte[0]);
        }
        return distroDataStorage.getDistroData(distroKey);
    }
    
    /**
     * Query all datum snapshot.
     *
     * @param type datum type
     * @return all datum snapshot
     */
    public DistroData onSnapshot(String type) {
        DistroDataStorage distroDataStorage = distroComponentHolder.findDataStorage(type);
        if (null == distroDataStorage) {
            Loggers.DISTRO.warn("[DISTRO] Can't find data storage for received key {}", type);
            return new DistroData(new DistroKey("snapshot", type), new byte[0]);
        }
        return distroDataStorage.getDatumSnapshot();
    }
}

```



#### 3.1 开启verify任务DistroVerifyTimedTask

```
@Override
public void run() {
    try {
    	//获取peer节点
        List<Member> targetServer = serverMemberManager.allMembersWithoutSelf();
        
        for (String each : distroComponentHolder.getDataStorageTypes()) {
        	//对dataStorage内的数据进行验证
        	//Nacos2.x版本，这里的type是“Nacos:Naming:v2:ClientData”；
        	//Nacos1.x版本，这里的type是“com.alibaba.nacos.naming.iplist.”
            verifyForDataStorage(each, targetServer);
        }
    } catch (Exception e) {
        Loggers.DISTRO.error("[DISTRO-FAILED] verify task failed.", e);
    }
}

private void verifyForDataStorage(String type, List<Member> targetServer) {
        DistroDataStorage dataStorage = distroComponentHolder.findDataStorage(type);
        
        List<DistroData> verifyData = dataStorage.getVerifyData();
        
        for (Member member : targetServer) {
            DistroTransportAgent agent = distroComponentHolder.findTransportAgent(type);
            //获取verifyData封装成DistroVerifyExecuteTask并提交
            executeTaskExecuteEngine.addTask(member.getAddress() + type,
                    new DistroVerifyExecuteTask(agent, verifyData, member.getAddress(), type));
        }
    }
```

```
public class DistroVerifyExecuteTask extends AbstractExecuteTask {
    //数据传输agent
    private final DistroTransportAgent transportAgent;
    //verify数据
    private final List<DistroData> verifyData;
    //目标server ip:port
    private final String targetServer;
    //资源类型
    private final String resourceType;
    
    @Override
    public void run() {
        for (DistroData each : verifyData) {
            try {
                if (transportAgent.supportCallbackTransport()) {
                	//2.0中发送带回调的请求
                    doSyncVerifyDataWithCallback(each);
                } else {
                    doSyncVerifyData(each);
                }
            } catch (Exception e) {
                Loggers.DISTRO
                        .error("[DISTRO-FAILED] verify data for type {} to {} failed.", resourceType, targetServer, e);
            }
        }
    }
}
```

#### 3.2 开启load任务DistroLoadDataTask

```java
DistroProtocol#startLoadTask
private void startLoadTask() {
        DistroCallback loadCallback = new DistroCallback() {
            @Override
            public void onSuccess() {
                //load任务成功，才设置DistroProtocol初始化成功
                isInitialized = true;
            }
            
            @Override
            public void onFailed(Throwable throwable) {
                isInitialized = false;
            }
        };
        GlobalExecutor.submitLoadDataTask(
                new DistroLoadDataTask(memberManager, distroComponentHolder, DistroConfig.getInstance(), loadCallback));
    }
```

```java
public class DistroLoadDataTask implements Runnable {
    
    private final ServerMemberManager memberManager;
    
    private final DistroComponentHolder distroComponentHolder;
    
    private final DistroConfig distroConfig;
    
    private final DistroCallback loadCallback;
    //保存每种type的load状态，Nacos2.x版本，这里的type是“Nacos:Naming:v2:ClientData”
    private final Map<String, Boolean> loadCompletedMap;
    
    @Override
    public void run() {
        try {
            load();
            if (!checkCompleted()) {
                //load失败，延迟30s后重试
                GlobalExecutor.submitLoadDataTask(this, distroConfig.getLoadDataRetryDelayMillis());
            } else {
                loadCallback.onSuccess();
                Loggers.DISTRO.info("[DISTRO-INIT] load snapshot data success");
            }
        } catch (Exception e) {
            loadCallback.onFailed(e);
            Loggers.DISTRO.error("[DISTRO-INIT] load snapshot data failed. ", e);
        }
    }
    
    private void load() throws Exception {
        //1.等待memberManager初始化完成
        while (memberManager.allMembersWithoutSelf().isEmpty()) {
            Loggers.DISTRO.info("[DISTRO-INIT] waiting server list init...");
            TimeUnit.SECONDS.sleep(1);
        }
        //2.等待distroComponentHolder初始化完成
        while (distroComponentHolder.getDataStorageTypes().isEmpty()) {
            Loggers.DISTRO.info("[DISTRO-INIT] waiting distro data storage register...");
            TimeUnit.SECONDS.sleep(1);
        }
        for (String each : distroComponentHolder.getDataStorageTypes()) {
            if (!loadCompletedMap.containsKey(each) || !loadCompletedMap.get(each)) {
                //3.从远程获取所有数据
                loadCompletedMap.put(each, loadAllDataSnapshotFromRemote(each));
            }
        }
    }
    
    private boolean loadAllDataSnapshotFromRemote(String resourceType) {
        DistroTransportAgent transportAgent = distroComponentHolder.findTransportAgent(resourceType);
        DistroDataProcessor dataProcessor = distroComponentHolder.findDataProcessor(resourceType);
        //遍历其他peer节点，拉取全量数据加载到本地内存
        for (Member each : memberManager.allMembersWithoutSelf()) {
            try {
                Loggers.DISTRO.info("[DISTRO-INIT] load snapshot {} from {}", resourceType, each.getAddress());
                DistroData distroData = transportAgent.getDatumSnapshot(each.getAddress());
                boolean result = dataProcessor.processSnapshot(distroData);
                Loggers.DISTRO
                        .info("[DISTRO-INIT] load snapshot {} from {} result: {}", resourceType, each.getAddress(),
                                result);
                if (result) {
                    distroComponentHolder.findDataStorage(resourceType).finishInitial();
                    return true;
                }
            } catch (Exception e) {
                Loggers.DISTRO.error("[DISTRO-INIT] load snapshot {} from {} failed.", resourceType, each.getAddress(), e);
            }
        }
        return false;
    }
}

```
