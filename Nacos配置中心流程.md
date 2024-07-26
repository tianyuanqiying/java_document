# Nacos配置中心

服务端配置变更，长轮训

- ConfigController
- PersistService
- ConfigDataChangeEvent
- AsyncNotifyService
- ServerMemberManager
- AsyncTask
- CommunicationController
- DumpService
- TaskManager
- DumpTask
- DumpProcessor =》NacosTaskProcessor
- DumpConfigHandler
- ConfigCacheService
- LocalDataChangeEvent
- LongPollingService
- DataChangeTask
- ClientLongPolling



客户端



### Nacos Config 配置文件的优先级

对于某一项配置、例如server.port , 若是多个配置文件中都存在，就存在顺序问题；

shared-nacos-config:共享配置

ext-nacos-config: 扩展配置

#{spring.application.name}  : 服务名配置

#{spring.application.name}.yaml:  服务名.yaml配置

#{spring.application.name}-#{spring.profiles.active}.yaml: 服务名-环境配置；





## Nacos加载配置文件流程

- SpringApplication#run : 启动方法
- SpringApplication#prepareEnvironment ：加载项目中的配置文件
  - 通过事件发布器发布ApplicationEnvironmentPreparedEvent时间；
    - ConfigFileApplicationListener监听该事件，触发后，读取本地配置文件；
      - 通过PropertiesPropertySourceLoader加载配置文件；
- SpringApplication#preparedContext 
  - ApplicationContextInitializer#initialize ： 监听器初始化
    - PropertySourceBootstrapConfiguration#initialize 
      - NacosPropertySourceLocator#locate
        - this.loadSharedConfiguration(composite) ： 加载共享配置
        - this.loadExtConfiguration(composite) ： 加载扩展配置
        - this.loadApplicationConfiguration(composite, dataIdPrefix, this.nacosConfigProperties, env)： 加载远程服务配置
          - loadNacosDataIfPresent(compositePropertySource, dataIdPrefix, nacosGroup, fileExtension, true)
            - 首先加载配置名为dataIdPrefix配置， 如：nacos-config；
          - loadNacosDataIfPresent(compositePropertySource, dataIdPrefix + "." + fileExtension, nacosGroup, fileExtension, true)
            - 其次加载配置名为dataIdPrefix + "." + fileExtension的配置,如nacos-config.yaml；
          - String dataId = dataIdPrefix + "-" + profile + "." + fileExtension;
            this.loadNacosDataIfPresent(compositePropertySource, dataId, nacosGroup, fileExtension, true);
            - 最后加载dataIdPrefix + "-" + profile + "." + fileExtension的配置，即nacos-config-dev.yaml
        - NacosConfigService.getConfig(dataId, group, timeout) :  加载共享、扩展，服务配置，底层调用ConfigService#getConfig， 通过Http请求调用Nacos服务；
          - LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant)
            - 远程Nacos配置 在本地存在缓存文件，首先从缓存文件中获取配置，防止Nacos宕机，出现故障；
          - ClientWorker#getServerConfig: 
            - agent.httpGet(Constants.CONFIG_CONTROLLER_PATH, null, params, agent.getEncode(), readTimeout) ： 发送HTTP请求；
              - URL：/v1/cs/configs, Method: GET 
            - saveSnapshot ：配置写入本地文件、防止Nacos宕机不可用；



### 概念

- PropertySource ：代表配置文件资源，以name-value的键值对形式存储配置文件内容
  - NacosPropertySource: 代表Nacos的配置文件资源

- CompositePropertySource： 代表合并的配置文件资源；内部属性propertySources代表批量配置文件；
  - 每次加载Nacos的远程配置文件资源NacosPropertySource后，调用addFirst方式propertySources的第一位； 因此，越靠后加载的文件，其配置优先级越高；
- ClientWorker ： Nacos Client 底层核心类
  - cacheMap  = Map<dataId+groupName+tenant, CacheData>： 存放Nacos配置监听器， 根据dataId+groupname+tenant进行分组；
- CacheData: ：存储监听器的容器；

​	

## 配置文件监听事件

SpringApplication#run

- listeners.running(context)  => 发布ApplicationReadyEvent事件
  - NacosContextRefresher#onApplicationEvent
    - registerNacosListener(propertySource.getGroup(), dataId) ： 根据dataId, groupName注册监听器， 监听配置变更；
      - Listener listener = new AbstractSharedListener() : 创建监听器；  <config-core>
      - configService.addListener(dataKey, groupKey, listener);
        - ClientWorker#addTenantListeners(String dataId, String group, List<? extends Listener> listeners);
          - 从**属性cacheMap获取CacheData，往CacheData的list插入监听器，用于长轮训**

当远程配置文件发生变更后，触发监听器

- AbstractSharedListener#innerReceive
  - nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo) ： 添加刷新记录；
  - appContext发布RefreshEvent事件
  - RefreshEventListener#onApplicationEvent(ApplicationEvent event)
    - ContextRefresher#refresh
      - refreshEnvironment ：新建SpringBoot容器，用来重新加载配置文件，加载完后，再和旧的配置进行比对，更新旧配置， 再销毁容器；
      - RefreshScope#refreshAll :  刷新RefreshScope中的Bean实例；
        - super.destroy() : 销毁GenericScope的cache缓存，下次会重新从BeanFactory中获取Bean
        - 发布RefreshScopeRefreshedEvent事件；





## Nacos客户端-服务端长轮训

spring-boot-starter-nacos-config的spring-factories指定NacosConfigBootstrapConfiguration自动配置类；

- NacosPropertySourceLocator ：用来加载Nacos远程配置文件；
- NacosConfigManager : 添加NacosConfigService类，实现ConfigService核心接口，发起配置请求；
  - NacosConfigService创建过程
    - 创建HttpAgent对象、用来发送Http请求；
    - 创建ClientWorker对象，用来实现长轮训， 创建两个定时线程池实现轮询；
      - 线程池属性executor :  核心线程数为1、只有一个线程；
      - 线程池属性executorService ： 核心线程数为CPU数；
      - 提交Runnable定时任务给executor， 延迟1毫秒，间隔10毫秒执行；



- Runnable#run 
  - ClientWorker#checkConfigInfo : 检查配置
  - int listenerSize = cacheMap.size() ： 获取监听器总数；
  - int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize())
    - 分页，每页3000个监听器；
  - executorService.execute(new LongPollingRunnable(i)) 
    - 往二次线程池提交长轮询任务、i为页数，每一页对应一个任务、即TaskId;



- LongPollingRunnable#run

  - checkLocalConfig(cacheData) 

    - 检查本地配置文件内容与内存中的配置内容不一致，则本地配置内容更新至内存中；

  - checkUpdateDataIds(cacheDatas, inInitializingCacheList)

    - checkUpdateConfigStr(sb.toString(), isInitializingCacheList) ： 注册配置监听器，超时时间为30s(长轮训)
      - params.put(Constants.PROBE_MODIFY_REQUEST, probeUpdateString)
        - 参数名：Listening-Configs， 值：监听的配置文件
      - headers.put("Long-Pulling-Timeout", "" + timeout)
        - 请求头设置长轮训超时时间
      - agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params, agent.getEncode(),readTimeoutMs)
      - URL：/v1/cs/configs/listener,  Method: POST,  timeout:30s, readTimeout:45s;
    - getServerConfig(dataId, group, tenant, 3000L)
      - HTTP请求NacosServer的获取配置接口：/v1/cs/configs， 获取配置后，存到本地文件；
    - cache.setContent(ct[0]) ；cache.setType(ct[1]) ： 更新内存的配置；
    - cacheData.checkListenerMd5() ： 每个监听器代表一个配置文件， 若内容Md5结果不一致，则触发监听器，更新容器Bean；
      - safeNotifyListener(dataId, group, content, type, md5, wrap)
        - Runnable#run
      - listener.receiveConfigInfo(contentTmp) :
            - listener#innerReceive(dataId, group, configInfo);
              - AbstractSharedListener#innerReceive ==><config-core>
    - executorService.execute(this) : 嵌套调用，继续执行；
    
    





### 服务端处理注册配置监听器

- ConfigController#listener
  - String probeModify = request.getParameter("Listening-Configs") : 获取监听的配置文件
  - Map clientMd5Map = MD5Util.getClientMd5Map(probeModify) ：
    - dataId+groupName+tenant为key,  配置内容MD5为value;
  - ConfigServletInner#doPollingConfig(request, response, clientMd5Map, probeModify.length())
    - LongPollingService.isSupportLongPolling(request)： 请求头包含Long-Pulling-Timeout， 则支持长轮训
    - 支持长轮训, 执行长轮训
      - longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize)
    - 不支持长轮询
      - MD5Util.compareMd5(request, response, clientMd5Map) ： 配置内容MD5值比较，获取变化的配置
      - response.addHeader(Constants.PROBE_MODIFY_RESPONSE_NEW, newResult) 
        - 响应头设置变更的配置key

执行长轮训

- longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize)

  - String str = req.getHeader(LongPollingService.LONG_POLLING_HEADER) 

    - 获取请求头Long-Pulling-Timeout、默认30s;

  - int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500) 

    - 获取延迟时间

  - long timeout = Math.max(10000, Long.parseLong(str) - delayTime);

    - 计算长轮训值、默认29.5s

  - List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);

    - 比较配置的md5值，获取变更的配置 

  - 若存在配置变更、changedGroups 不为空

    - generateResponse(req, rsp, changedGroups) 
      - response.getWriter().println(respString)： 响应变更数据

  - 若不存在配置变更、等待29.5s在响应；

    - ConfigExecutor.executeLongPolling(new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));

    - ClientLongPolling#run

      - ConfigExecutor.scheduleLongPolling(new Runnable(), timeoutTime, TimeUnit.MILLISECONDS);
        - 提交任务给线程池、延迟29.5s执行, 会从allSubs阻塞队列获取长轮训任务；
      - allSubs.add(this) ： 长轮训任务放入阻塞队列allSubs

    - Runnable#run : 

      - allSubs.remove(ClientLongPolling.this) ： 阻塞队列 中获取长轮训任务
      - MD5Util.compareMd5((HttpServletRequest) asyncContext.getRequest(),
                        (HttpServletResponse) asyncContext.getResponse(), clientMd5Map);
        - 比较配置内容MD5值、获取变更配置
      - sendResponse(changedGroups)
        - 发送变更配置给客户端

      

ClientLongPolling提交给线程池执行：

1. 创建调度任务、延迟29.5s;
2. 将自身ClientLongPolling实例放入阻塞队列allSubs；
3. 延迟29.5s后，执行调度任务、从allSubs阻塞队列获取并移除自身ClientLongPolling实例
4. 服务端检查判断客户端请求的groupKeys是否发生变更，结果写入response发送个客户端;





### 控制台修改配置发布流程

- ConfigController#publishConfig

  - persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, true)

    -  配置了Mysql，实现类为ExternalStoragePersistServiceImpl，默认为EmbeddedStoragePersistServiceImpl

    - 若是新增操作：ExternalStoragePersistServiceImpl#addConfigInfo(srcIp, srcUser, configInfo, time, configAdvanceInfo, notify) ：数据库插入记录

      - addConfigInfoAtomic(-1, srcIp, srcUser, configInfo, time, configAdvanceInfo)

        - ```mysql
          INSERT INTO config_info(data_id,group_id,tenant_id,app_name,content,md5,src_ip,src_user,gmt_create,"
                  + "gmt_modified,c_desc,c_use,effect,type,c_schema) VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)
          ```

        - 往ConfigInfo表插入一条记录

      - addConfigTagsRelation(configId, configTags, configInfo.getDataId(), configInfo.getGroup(), configInfo.getTenant())

        - ```mysql
          INSERT INTO config_tags_relation
          (id,tag_name,tag_type,data_id,group_id,tenant_id) VALUES(?,?,?,?,?,?)
          ```

        - 往config_tags_relation表中插入记录

      - insertConfigHistoryAtomic(0, configInfo, srcIp, srcUser, time, "I")

        - ```mysql
          INSERT INTO his_config_info (id,data_id,group_id,tenant_id,app_name,content,md5,src_ip,src_user,gmt_modified,op_type) VALUES(?,?,?,?,?,?,?,?,?,?,?)
          ```

        - 往历史表his_config_info中插入历史记录

        

    - 若是更新操作：ExternalStoragePersistServiceImpl#updateConfigInfo(configInfo, srcIp, srcUser, time, configAdvanceInfo, notify);

      - findConfigInfo(configInfo.getDataId(), configInfo.getGroup(),configInfo.getTenant())

        - ```mysql
          SELECT ID,data_id,group_id,tenant_id,app_name,content,md5,type FROM config_info WHERE data_id=? AND group_id=? AND tenant_id=?
          ```

        - 根据dataId, groupId, tenantId查询配置记录

      - updateConfigInfoAtomic(configInfo, srcIp, srcUser, time, configAdvanceInfo);

        - ```mysql
          jt.update("UPDATE config_info SET content=?, md5 = ?, src_ip=?,src_user=?,gmt_modified=?,app_name=?,c_desc=?,c_use=?,effect=?,type=?,c_schema=?  WHERE data_id=? AND group_id=? AND tenant_id=?", configInfo.getContent(), md5Tmp, srcIp, srcUser, time, appNameTmp, desc, use, effect, type, schema, configInfo.getDataId(),configInfo.getGroup(),
          ```

        - 根据dataId, groupId, tenantId更新配置记录

      - removeTagByIdAtomic(oldConfigInfo.getId()) 

        - ```mysql
          DELETE FROM config_tags_relation WHERE id=?
          ```

        - 根据ID删除旧的tag表

      - addConfigTagsRelation(oldConfigInfo.getId(), configTags, configInfo.getDataId(),
                configInfo.getGroup(), configInfo.getTenant())

        - ```mysql
          INSERT INTO config_tags_relation
          (id,tag_name,tag_type,data_id,group_id,tenant_id) VALUES(?,?,?,?,?,?)
          ```

        - 插入新的tag记录；

      - insertConfigHistoryAtomic(oldConfigInfo.getId(), oldConfigInfo, srcIp, srcUser, time, "U");

        - ```mysql
          INSERT INTO his_config_info (id,data_id,group_id,tenant_id,app_name,content,md5,src_ip,src_user,gmt_modified,op_type) VALUES(?,?,?,?,?,?,?,?,?,?,?)
          ```

  - ConfigChangePublisher.notifyConfigChange(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));

  - AsyncNotifyService构造方法创建监听器Subscriber， 监听ConfigDataChangeEvent
  - Subscriber#onEvent
    - Collection<Member> ipList = memberManager.allMembers() ：获取所有Nacos结点，也包括自己
    - queue.add(new NotifySingleTask(dataId, group, tenant, tag, dumpTs, member.getAddress(),
              evt.isBeta));
    - ConfigExecutor.executeAsyncNotify(new AsyncTask(nacosAsyncRestTemplate, queue))
    - AsyncTask#run
      - NotifySingleTask task = queue.poll() ： 从队列中取出任务
      -  memberManager.isUnHealth(targetIp) ： 检查Nacos节点健康状态
        - 若nacos状态为不健康，则将任务延迟执行
        - ConfigExecutor.scheduleAsyncNotify(asyncTask, delay, TimeUnit.MILLISECONDS)
        - 若nacos状态为健康、则同步变更配置；
        - restTemplate.get(task.url, header, Query.EMPTY, String.class, new AsyncNotifyCallBack(task));
          - url:/communication/dataChanged?dataId={2}&group={3},  method:get， 回调：AsyncNotifyCallBack
          - AsyncNotifyCallBack#onReceive
            - 若调用失败、则延迟重复调用；
            - ConfigExecutor.scheduleAsyncNotify(asyncTask, delay, TimeUnit.MILLISECONDS);



Nacos结点处理配置变更；

- CommunicationController#notifyConfigInfo
  - dumpService.dump(dataId, group, tenant, tag, lastModifiedTs, handleIp)
  - dumpTaskMgr.addTask(taskKey, new DumpTask(groupKey, lastModified, handleIp, isBeta));
    - tasks.put(key, newTask)， 类型为**DumpTask**
    - 往TaskManager队列中新增任务，父类NacosDelayTaskExecuteEngine存在tasks属性；



NacosDelayTaskExecuteEngine构造方法中， 提交任务给线程池执行；

```java
processingExecutor.scheduleWithFixedDelay(new ProcessRunnable(), processInterval, processInterval, TimeUnit.MILLISECONDS);
```



- ProcessRunnable#run
  - processTasks();
  - AbstractDelayTask task = removeTask(taskKey)
    - 从tasks容器中取出任务
  - processor.process(task)
    - 任务类型为DumpTask， 处理器类型为DumpProcessor
  - DumpConfigHandler.configDump(build.build());
    - result = ConfigCacheService.dump(dataId, group, namespaceId, content, lastModified, type);
      - String md5 = MD5Utils.md5Hex(content, Constants.ENCODE) 
        - 获取发布的配置内容MD5值；
      - updateMd5(groupKey, md5, lastModifiedTs);
        - NotifyCenter.publishEvent(new LocalDataChangeEvent(groupKey))
          - 发布本地数据变更事件

LongPollingService构造方法中，注册了监听器、监听LocalDataChangeEvent事件；

```java
public LongPollingService() {
    // Register LocalDataChangeEvent to NotifyCenter.
    NotifyCenter.registerToPublisher(LocalDataChangeEvent.class, NotifyCenter.ringBufferSize);
    NotifyCenter.registerSubscriber(new Subscriber() {
        @Override
        public void onEvent(Event event) {
            if (isFixedPolling()) {
                // Ignore.
            } else {
                if (event instanceof LocalDataChangeEvent) {
                    LocalDataChangeEvent evt = (LocalDataChangeEvent) event;
                    ConfigExecutor.executeLongPolling(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));
                }
            }
        }
    });
    
}
```

LongPollingService<init> =>Subscriber # onEvent

- ConfigExecutor.executeLongPolling(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));

  - 提交数据变更任务到线程池中；

- DataChangeTask#run

  - Iterator<ClientLongPolling> iter = allSubs.iterator()

    - 阻塞队列中获取长轮训任务、

  - clientSub.clientMd5Map.containsKey(groupKey) : 检查客户端是否包含当前变更的key

  - clientSub.sendResponse(Arrays.asList(groupKey)) : 要给客户端发送变更的key;

    - generateResponse(changedGroups);

      - response.getWriter().println(respString) : 写入响应

      - asyncContext.complete() ：阻塞结束、响应

        

      

### 服务端获取配置接口

- ConfigController#getConfig
  - ConfigServletInner#doGetConfig(request, response, dataId, group, tenant, tag, clientIp)
    - file = DiskUtil.targetFile(dataId, group, tenant);
    - file = DiskUtil.targetTagFile(dataId, group, tenant, tag);
      - 从文件中获取配置， 而不是数据库、减少DB依赖；
    - FileInputStream fis = = new FileInputStream(file);
    - fis.getChannel() .transferTo(0L, fis.getChannel().size(), Channels.newChannel(response.getOutputStream()));
      - 写入响应流，返回；

