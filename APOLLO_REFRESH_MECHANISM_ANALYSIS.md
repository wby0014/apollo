# Apollo 定时刷新与长轮询机制深度解析

## 一、概述

Apollo 采用**双重保障机制**来确保配置的实时性：
1. **定时刷新（Periodic Refresh）**：作为兜底机制，定期拉取配置
2. **长轮询（Long Polling）**：作为主要机制，实现配置的准实时推送

这种设计既保证了配置的实时性，又避免了频繁轮询带来的性能问题。

---

## 二、定时刷新机制（Periodic Refresh）

### 2.1 实现位置

定时刷新在 `RemoteConfigRepository` 中实现，每个 namespace 都有自己独立的定时刷新任务。

### 2.2 源码分析

#### 2.2.1 初始化定时任务

```148:163:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    private void schedulePeriodicRefresh() {
        logger.debug("Schedule periodic refresh with interval: {} {}", m_configUtil.getRefreshInterval(), m_configUtil.getRefreshIntervalTimeUnit());
        // 创建定时任务，定时刷新配置
        m_executorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                // 【TODO 6001】Tracer 日志
                Tracer.logEvent("Apollo.ConfigService", String.format("periodicRefresh: %s", m_namespace));
                logger.debug("refresh config for namespace: {}", m_namespace);
                // 尝试同步配置
                trySync();
                // 【TODO 6001】Tracer 日志
                Tracer.logEvent("Apollo.Client.Version", Apollo.VERSION);
            }
        }, m_configUtil.getRefreshInterval(), m_configUtil.getRefreshInterval(), m_configUtil.getRefreshIntervalTimeUnit());
    }
```

**关键点**：
- 使用 `ScheduledExecutorService.scheduleAtFixedRate()` 创建固定频率的定时任务
- 默认刷新间隔由 `ConfigUtil.getRefreshInterval()` 决定（通常为 5 分钟）
- 每个 `RemoteConfigRepository` 实例共享同一个线程池（静态变量 `m_executorService`）

#### 2.2.2 配置同步逻辑

```166:198:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    @Override
    protected synchronized void sync() {
        // 【TODO 6001】Tracer 日志
        Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "syncRemoteConfig");
        try {
            // 获得缓存的 ApolloConfig 对象
            ApolloConfig previous = m_configCache.get();
            // 从 Config Service 加载 ApolloConfig 对象
            ApolloConfig current = loadApolloConfig();

            // reference equals means HTTP 304
            // 若不相等，说明更新了，设置到缓存中
            if (previous != current) {
                logger.debug("Remote Config refreshed!");
                // 设置到缓存
                m_configCache.set(current);
                // 发布 Repository 的配置发生变化，触发对应的监听器们
                super.fireRepositoryChange(m_namespace, this.getConfig());
            }
            // 【TODO 6001】Tracer 日志
            if (current != null) {
                Tracer.logEvent(String.format("Apollo.Client.Configs.%s", current.getNamespaceName()), current.getReleaseKey());
            }
            // 【TODO 6001】Tracer 日志
            transaction.setStatus(Transaction.SUCCESS);
        } catch (Throwable ex) {
            // 【TODO 6001】Tracer 日志
            transaction.setStatus(ex);
            throw ex;
        } finally {
            // 【TODO 6001】Tracer 日志
            transaction.complete();
        }
    }
```

**同步流程**：
1. 获取当前缓存的配置（`previous`）
2. 从 Config Service 拉取最新配置（`current`）
3. **对象引用比较**：如果 `previous != current`，说明配置有更新
4. 更新缓存并触发配置变更事件

#### 2.2.3 HTTP 304 优化

```206:313:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    private ApolloConfig loadApolloConfig() {
        // 限流
        if (!m_loadConfigRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
            // wait at most 5 seconds
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
            }
        }
        // 获得 appId cluster dataCenter 配置信息
        String appId = m_configUtil.getAppId();
        String cluster = m_configUtil.getCluster();
        String dataCenter = m_configUtil.getDataCenter();
        Tracer.logEvent("Apollo.Client.ConfigMeta", STRING_JOINER.join(appId, cluster, m_namespace));
        // 计算重试次数
        int maxRetries = m_configNeedForceRefresh.get() ? 2 : 1;
        long onErrorSleepTime = 0; // 0 means no sleep
        Throwable exception = null;
        // 获得所有的 Config Service 的地址
        List<ServiceDTO> configServices = getConfigServices();
        String url = null;
        // 循环读取配置重试次数直到成功。每一次，都会循环所有的 ServiceDTO 数组。
        for (int i = 0; i < maxRetries; i++) {
            // 随机所有的 Config Service 的地址
            List<ServiceDTO> randomConfigServices = Lists.newLinkedList(configServices);
            Collections.shuffle(randomConfigServices);
            // 优先访问通知配置变更的 Config Service 的地址。并且，获取到时，需要置空，避免重复优先访问。
            // Access the server which notifies the client first
            if (m_longPollServiceDto.get() != null) {
                randomConfigServices.add(0, m_longPollServiceDto.getAndSet(null));
            }
            // 循环所有的 Config Service 的地址
            for (ServiceDTO configService : randomConfigServices) {
                // sleep 等待，下次从 Config Service 拉取配置
                if (onErrorSleepTime > 0) {
                    logger.warn("Load config failed, will retry in {} {}. appId: {}, cluster: {}, namespaces: {}", onErrorSleepTime, m_configUtil.getOnErrorRetryIntervalTimeUnit(), appId, cluster, m_namespace);
                    try {
                        m_configUtil.getOnErrorRetryIntervalTimeUnit().sleep(onErrorSleepTime);
                    } catch (InterruptedException e) {
                        //ignore
                    }
                }
                // 组装查询配置的地址
                url = assembleQueryConfigUrl(configService.getHomepageUrl(), appId, cluster, m_namespace, dataCenter, m_remoteMessages.get(), m_configCache.get());

                logger.debug("Loading config from {}", url);
                // 创建 HttpRequest 对象
                HttpRequest request = new HttpRequest(url);

                // 【TODO 6001】Tracer 日志
                Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "queryConfig");
                transaction.addData("Url", url);
                try {
                    // 发起请求，返回 HttpResponse 对象
                    HttpResponse<ApolloConfig> response = m_httpUtil.doGet(request, ApolloConfig.class);
                    // 设置 m_configNeedForceRefresh = false
                    m_configNeedForceRefresh.set(false);
                    // 标记成功
                    m_loadConfigFailSchedulePolicy.success();

                    // 【TODO 6001】Tracer 日志
                    transaction.addData("StatusCode", response.getStatusCode());
                    transaction.setStatus(Transaction.SUCCESS);

                    // 无新的配置，直接返回缓存的 ApolloConfig 对象
                    if (response.getStatusCode() == 304) {
                        logger.debug("Config server responds with 304 HTTP status code.");
                        return m_configCache.get();
                    }

                    // 有新的配置，进行返回新的 ApolloConfig 对象
                    ApolloConfig result = response.getBody();
                    logger.debug("Loaded config for {}: {}", m_namespace, result);
                    return result;
                } catch (ApolloConfigStatusCodeException ex) {
                    ApolloConfigStatusCodeException statusCodeException = ex;
                    // 若返回的状态码是 404 ，说明查询配置的 Config Service 不存在该 Namespace 。
                    // config not found
                    if (ex.getStatusCode() == 404) {
                        String message = String.format("Could not find config for namespace - appId: %s, cluster: %s, namespace: %s, " +
                                "please check whether the configs are released in Apollo!", appId, cluster, m_namespace);
                        statusCodeException = new ApolloConfigStatusCodeException(ex.getStatusCode(), message);
                    }
                    // 【TODO 6001】Tracer 日志
                    Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(statusCodeException));
                    transaction.setStatus(statusCodeException);
                    // 设置最终的异常
                    exception = statusCodeException;
                } catch (Throwable ex) {
                    // 【TODO 6001】Tracer 日志
                    Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
                    transaction.setStatus(ex);
                    // 设置最终的异常
                    exception = ex;
                } finally {
                    // 【TODO 6001】Tracer 日志
                    transaction.complete();
                }
                // 计算延迟时间
                // if force refresh, do normal sleep, if normal config load, do exponential sleep
                onErrorSleepTime = m_configNeedForceRefresh.get() ? m_configUtil.getOnErrorRetryInterval() : m_loadConfigFailSchedulePolicy.fail();
            }

        }
        // 若查询配置失败，抛出 ApolloConfigException 异常
        String message = String.format("Load Apollo Config failed - appId: %s, cluster: %s, namespace: %s, url: %s", appId, cluster, m_namespace, url);
        throw new ApolloConfigException(message, exception);
    }
```

**关键优化点**：

1. **ReleaseKey 机制**：
   - 请求 URL 中携带 `releaseKey`（当前配置的版本号）
   - 如果服务端配置未变更，返回 **HTTP 304 Not Modified**
   - 客户端直接使用缓存，**无需传输配置内容**，节省带宽

2. **URL 组装**：
```315:349:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    // 组装查询配置的地址
    String assembleQueryConfigUrl(String uri, String appId, String cluster, String namespace,
                                  String dataCenter, ApolloNotificationMessages remoteMessages, ApolloConfig previousConfig) {
        String path = "configs/%s/%s/%s"; // /configs/{appId}/{clusterName}/{namespace:.+}
        List<String> pathParams = Lists.newArrayList(pathEscaper.escape(appId), pathEscaper.escape(cluster), pathEscaper.escape(namespace));
        Map<String, String> queryParams = Maps.newHashMap();
        // releaseKey
        if (previousConfig != null) {
            queryParams.put("releaseKey", queryParamEscaper.escape(previousConfig.getReleaseKey()));
        }
        // dataCenter
        if (!Strings.isNullOrEmpty(dataCenter)) {
            queryParams.put("dataCenter", queryParamEscaper.escape(dataCenter));
        }
        // ip
        String localIp = m_configUtil.getLocalIp();
        if (!Strings.isNullOrEmpty(localIp)) {
            queryParams.put("ip", queryParamEscaper.escape(localIp));
        }
        // messages
        if (remoteMessages != null) {
            queryParams.put("messages", queryParamEscaper.escape(gson.toJson(remoteMessages)));
        }
        // 格式化 URL
        String pathExpanded = String.format(path, pathParams.toArray());
        // 拼接 Query String
        if (queryParams.isEmpty()) {
            pathExpanded += "?" + MAP_JOINER.join(queryParams);
        }
        // 拼接最终的请求 URL
        if (!uri.endsWith("/")) {
            uri += "/";
        }
        return uri + pathExpanded;
    }
```

3. **限流保护**：
   - 使用 `RateLimiter` 控制请求频率
   - 防止定时任务过多导致服务端压力过大

4. **负载均衡**：
   - 随机打乱 Config Service 列表
   - 优先访问长轮询通知的 Config Service（`m_longPollServiceDto`）

### 2.3 为什么需要定时刷新？

**设计原因**：

1. **兜底机制**：长轮询可能因为网络问题、服务端重启等原因失败，定时刷新确保即使长轮询失效，配置也能在合理时间内更新
2. **容错性**：如果长轮询服务异常，定时刷新仍然可以工作
3. **配置一致性**：定期校验配置，确保客户端配置与服务端一致

**缺点**：
- 延迟较高（默认 5 分钟）
- 可能产生不必要的请求（即使配置未变更）

---

## 三、长轮询机制（Long Polling）

### 3.1 什么是长轮询？

长轮询是**轮询的优化版本**：
- **普通轮询**：客户端每隔一段时间（如 1 秒）发起一次请求，无论是否有数据
- **长轮询**：客户端发起请求后，服务端**保持连接**，直到有数据变更或超时才返回

**优势**：
- 实时性更好：配置变更后立即返回
- 减少无效请求：无变更时只返回一次 304，而不是多次轮询

### 3.2 实现架构

Apollo 的长轮询采用**单例模式**，所有 namespace 共享一个长轮询服务：

```107:122:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    public RemoteConfigLongPollService() {
        m_longPollFailSchedulePolicyInSecond = new ExponentialSchedulePolicy(1, 120); //in second
        m_longPollingStopped = new AtomicBoolean(false);
        m_longPollingService = Executors.newSingleThreadExecutor(ApolloThreadFactory.create("RemoteConfigLongPollService", true));
        m_longPollStarted = new AtomicBoolean(false);
        m_longPollNamespaces = Multimaps.synchronizedSetMultimap(HashMultimap.<String, RemoteConfigRepository>create());
        m_notifications = Maps.newConcurrentMap();
        m_remoteNotificationMessages = Maps.newConcurrentMap();
        m_responseType = new TypeToken<List<ApolloConfigNotification>>() {
        }.getType();
        gson = new Gson();
        m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
        m_httpUtil = ApolloInjector.getInstance(HttpUtil.class);
        m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
        m_longPollRateLimiter = RateLimiter.create(m_configUtil.getLongPollQPS());
    }
```

**关键数据结构**：
- `m_longPollNamespaces`：Multimap，存储每个 namespace 对应的 `RemoteConfigRepository` 列表
- `m_notifications`：ConcurrentMap，存储每个 namespace 的最新 notificationId
- `m_remoteNotificationMessages`：存储远程通知消息

### 3.3 注册流程

当创建 `RemoteConfigRepository` 时，会注册到长轮询服务：

```354:356:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    private void scheduleLongPollingRefresh() {
        remoteConfigLongPollService.submit(m_namespace, this);
    }
```

```131:141:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    public boolean submit(String namespace, RemoteConfigRepository remoteConfigRepository) {
        // 添加到 m_longPollNamespaces 中
        boolean added = m_longPollNamespaces.put(namespace, remoteConfigRepository);
        // 添加到 m_notifications 中
        m_notifications.putIfAbsent(namespace, INIT_NOTIFICATION_ID);
        // 若未启动长轮询定时任务，进行启动
        if (!m_longPollStarted.get()) {
            startLongPolling();
        }
        return added;
    }
```

### 3.4 长轮询核心逻辑

#### 3.4.1 启动长轮询

```146:184:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    private void startLongPolling() {
        // CAS 设置长轮询任务已经启动。若已经启动，不重复启动。
        if (!m_longPollStarted.compareAndSet(false, true)) {
            //already started
            return;
        }
        try {
            // 获得 appId cluster dataCenter 配置信息
            final String appId = m_configUtil.getAppId();
            final String cluster = m_configUtil.getCluster();
            final String dataCenter = m_configUtil.getDataCenter();
            // 获得长轮询任务的初始化延迟时间，单位毫秒。
            final long longPollingInitialDelayInMills = m_configUtil.getLongPollingInitialDelayInMills();
            // 提交长轮询任务。该任务会持续且循环执行。
            m_longPollingService.submit(new Runnable() {
                @Override
                public void run() {
                    // 初始等待
                    if (longPollingInitialDelayInMills > 0) {
                        try {
                            logger.debug("Long polling will start in {} ms.", longPollingInitialDelayInMills);
                            TimeUnit.MILLISECONDS.sleep(longPollingInitialDelayInMills);
                        } catch (InterruptedException e) {
                            //ignore
                        }
                    }
                    // 执行长轮询
                    doLongPollingRefresh(appId, cluster, dataCenter);
                }
            });
        } catch (Throwable ex) {
            // 设置 m_longPollStarted 为 false
            m_longPollStarted.set(false);
            // 【TODO 6001】Tracer 日志
            ApolloConfigException exception = new ApolloConfigException("Schedule long polling refresh failed", ex);
            Tracer.logError(exception);
            logger.warn(ExceptionUtil.getDetailMessage(exception));
        }
    }
```

**关键点**：
- 使用 **CAS（Compare-And-Swap）** 确保只启动一次
- 支持初始延迟，避免应用启动时立即发起请求

#### 3.4.2 长轮询循环

```197:277:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    private void doLongPollingRefresh(String appId, String cluster, String dataCenter) {
        final Random random = new Random();
        ServiceDTO lastServiceDto = null;
        // 循环执行，直到停止或线程中断
        while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
            // 限流
            if (!m_longPollRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
                // wait at most 5 seconds
                try {
                    TimeUnit.SECONDS.sleep(5);
                } catch (InterruptedException e) {
                }
            }
            // 【TODO 6001】Tracer 日志
            Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "pollNotification");
            String url = null;
            try {
                // 获得 Config Service 的地址
                if (lastServiceDto == null) {
                    // 获得所有的 Config Service 的地址
                    List<ServiceDTO> configServices = getConfigServices();
                    lastServiceDto = configServices.get(random.nextInt(configServices.size()));
                }
                // 组装长轮询通知变更的地址
                url = assembleLongPollRefreshUrl(lastServiceDto.getHomepageUrl(), appId, cluster, dataCenter, m_notifications);

                logger.debug("Long polling from {}", url);
                // 创建 HttpRequest 对象，并设置超时时间
                HttpRequest request = new HttpRequest(url);
                request.setReadTimeout(LONG_POLLING_READ_TIMEOUT);

                // 【TODO 6001】Tracer 日志
                transaction.addData("Url", url);

                // 发起请求，返回 HttpResponse 对象
                final HttpResponse<List<ApolloConfigNotification>> response = m_httpUtil.doGet(request, m_responseType);
                logger.debug("Long polling response: {}, url: {}", response.getStatusCode(), url);

                // 有新的通知，刷新本地的缓存
                if (response.getStatusCode() == 200 && response.getBody() != null) {
                    // 更新 m_notifications
                    updateNotifications(response.getBody());
                    // 更新 m_remoteNotificationMessages
                    updateRemoteNotifications(response.getBody());
                    // 【TODO 6001】Tracer 日志
                    transaction.addData("Result", response.getBody().toString());
                    // 通知对应的 RemoteConfigRepository 们
                    notify(lastServiceDto, response.getBody());
                }

                // 无新的通知，重置连接的 Config Service 的地址，下次请求不同的 Config Service ，实现负载均衡。
                // try to load balance
                if (response.getStatusCode() == 304 && random.nextBoolean()) { // 随机
                    lastServiceDto = null;
                }
                // 标记成功
                m_longPollFailSchedulePolicyInSecond.success();
                // 【TODO 6001】Tracer 日志
                transaction.addData("StatusCode", response.getStatusCode());
                transaction.setStatus(Transaction.SUCCESS);
            } catch (Throwable ex) {
                // 重置连接的 Config Service 的地址，下次请求不同的 Config Service
                lastServiceDto = null;
                // 【TODO 6001】Tracer 日志
                Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
                transaction.setStatus(ex);
                // 标记失败，计算下一次延迟执行时间
                long sleepTimeInSecond = m_longPollFailSchedulePolicyInSecond.fail();
                logger.warn("Long polling failed, will retry in {} seconds. appId: {}, cluster: {}, namespaces: {}, long polling url: {}, reason: {}",
                        sleepTimeInSecond, appId, cluster, assembleNamespaces(), url, ExceptionUtil.getDetailMessage(ex));
                // 等待一定时间，下次失败重试
                try {
                    TimeUnit.SECONDS.sleep(sleepTimeInSecond);
                } catch (InterruptedException ie) {
                    //ignore
                }
            } finally {
                transaction.complete();
            }
        }
    }
```

**核心流程**：

1. **无限循环**：直到应用停止或线程中断
2. **限流控制**：使用 `RateLimiter` 控制请求频率
3. **负载均衡**：随机选择 Config Service
4. **超时设置**：`LONG_POLLING_READ_TIMEOUT = 90秒`（比服务端 60 秒长）
5. **响应处理**：
   - **200**：有配置变更，立即处理
   - **304**：无变更，继续下一轮长轮询
6. **失败重试**：使用指数退避策略

#### 3.4.3 URL 组装

```355:377:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    // 组装长轮询通知变更的地址
    String assembleLongPollRefreshUrl(String uri, String appId, String cluster, String dataCenter, Map<String, Long> notificationsMap) {
        Map<String, String> queryParams = Maps.newHashMap();
        queryParams.put("appId", queryParamEscaper.escape(appId));
        queryParams.put("cluster", queryParamEscaper.escape(cluster));
        // notifications
        queryParams.put("notifications", queryParamEscaper.escape(assembleNotifications(notificationsMap)));
        // dataCenter
        if (!Strings.isNullOrEmpty(dataCenter)) {
            queryParams.put("dataCenter", queryParamEscaper.escape(dataCenter));
        }
        // ip
        String localIp = m_configUtil.getLocalIp();
        if (!Strings.isNullOrEmpty(localIp)) {
            queryParams.put("ip", queryParamEscaper.escape(localIp));
        }
        // 创建 Query String
        String params = MAP_JOINER.join(queryParams);
        // 拼接 URL
        if (!uri.endsWith("/")) {
            uri += "/";
        }
        return uri + "notifications/v2?" + params;
    }
```

**关键参数**：
- `notifications`：JSON 格式，包含所有 namespace 的 notificationId
  ```json
  [
    {"namespaceName": "application", "notificationId": 123},
    {"namespaceName": "TEST1.apollo", "notificationId": 456}
  ]
  ```

#### 3.4.4 通知处理

当长轮询收到配置变更通知时：

```279:306:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    private void notify(ServiceDTO lastServiceDto, List<ApolloConfigNotification> notifications) {
        if (notifications == null || notifications.isEmpty()) {
            return;
        }
        // 循环 ApolloConfigNotification
        for (ApolloConfigNotification notification : notifications) {
            String namespaceName = notification.getNamespaceName(); // Namespace 的名字
            // 创建 RemoteConfigRepository 数组，避免并发问题
            // create a new list to avoid ConcurrentModificationException
            List<RemoteConfigRepository> toBeNotified = Lists.newArrayList(m_longPollNamespaces.get(namespaceName));
            // 因为 .properties 在默认情况下被过滤掉，所以我们需要检查是否有监听器。若有，添加到 RemoteConfigRepository 数组
            // since .properties are filtered out by default, so we need to check if there is any listener for it
            toBeNotified.addAll(m_longPollNamespaces.get(String.format("%s.%s", namespaceName, ConfigFileFormat.Properties.getValue())));
            // 获得远程的 ApolloNotificationMessages 对象，并克隆
            ApolloNotificationMessages originalMessages = m_remoteNotificationMessages.get(namespaceName);
            ApolloNotificationMessages remoteMessages = originalMessages == null ? null : originalMessages.clone();
            // 循环 RemoteConfigRepository ，进行通知
            for (RemoteConfigRepository remoteConfigRepository : toBeNotified) {
                try {
                    // 进行通知
                    remoteConfigRepository.onLongPollNotified(lastServiceDto, remoteMessages);
                } catch (Throwable ex) {
                    // 【TODO 6001】Tracer 日志
                    Tracer.logError(ex);
                }
            }
        }
    }
```

**通知流程**：
1. 遍历所有变更的 namespace
2. 找到对应的 `RemoteConfigRepository` 列表
3. 调用 `onLongPollNotified()` 方法

#### 3.4.5 触发配置同步

```364:381:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    public void onLongPollNotified(ServiceDTO longPollNotifiedServiceDto, ApolloNotificationMessages remoteMessages) {
        // 设置长轮询到配置更新的 Config Service 。下次同步配置时，优先读取该服务
        m_longPollServiceDto.set(longPollNotifiedServiceDto);
        // 设置 m_remoteMessages
        m_remoteMessages.set(remoteMessages);
        // 提交同步任务
        m_executorService.submit(new Runnable() {

            @Override
            public void run() {
                // 设置 m_configNeedForceRefresh 为 true
                m_configNeedForceRefresh.set(true);
                // 尝试同步配置
                trySync();
            }

        });
    }
```

**关键点**：
- 保存通知的 Config Service，下次同步时优先访问
- 设置 `m_configNeedForceRefresh = true`，强制刷新（重试 2 次）
- 异步执行同步任务，不阻塞长轮询线程

### 3.5 为什么使用长轮询？

#### 3.5.1 对比普通轮询

| 方式 | 实时性 | 服务端压力 | 网络开销 |
|------|--------|-----------|----------|
| **普通轮询**（1秒间隔） | 延迟 0-1秒 | 高（每秒 N 个请求） | 高 |
| **普通轮询**（5秒间隔） | 延迟 0-5秒 | 中 | 中 |
| **长轮询**（90秒超时） | 延迟 0-90秒（通常<1秒） | 低（无变更时几乎无请求） | 低 |

#### 3.5.2 设计优势

1. **实时性好**：
   - 配置变更后，服务端立即返回，客户端在 1 秒内感知
   - 比定时刷新（5 分钟）快得多

2. **服务端压力小**：
   - 无变更时，90 秒内只返回一次 304
   - 相当于将请求频率从每秒 N 次降低到每 90 秒 1 次

3. **网络开销低**：
   - 304 响应体很小，几乎不占用带宽
   - 只在有变更时才传输配置内容

4. **支持多 namespace**：
   - 一次长轮询请求可以监听多个 namespace
   - 减少连接数

#### 3.5.3 为什么超时时间设为 90 秒？

```java
// 90 seconds, should be longer than server side's long polling timeout, which is now 60 seconds
private static final int LONG_POLLING_READ_TIMEOUT = 90 * 1000;
```

**原因**：
- 服务端长轮询超时为 60 秒
- 客户端设置为 90 秒，确保能收到服务端的正常响应（200 或 304）
- 如果客户端超时时间 < 服务端，可能在服务端返回前就超时，导致无效请求

---

## 四、两种机制的协作

### 4.1 初始化流程

```112:131:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java
    public RemoteConfigRepository(String namespace) {
        m_namespace = namespace;
        m_configCache = new AtomicReference<>();
        m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
        m_httpUtil = ApolloInjector.getInstance(HttpUtil.class);
        m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
        remoteConfigLongPollService = ApolloInjector.getInstance(RemoteConfigLongPollService.class);
        m_longPollServiceDto = new AtomicReference<>();
        m_remoteMessages = new AtomicReference<>();
        m_loadConfigRateLimiter = RateLimiter.create(m_configUtil.getLoadConfigQPS());
        m_configNeedForceRefresh = new AtomicBoolean(true);
        m_loadConfigFailSchedulePolicy = new ExponentialSchedulePolicy(m_configUtil.getOnErrorRetryInterval(), m_configUtil.getOnErrorRetryInterval() * 8);
        gson = new Gson();
        // 尝试同步配置
        super.trySync();
        // 初始化定时刷新配置的任务
        this.schedulePeriodicRefresh();
        // 注册自己到 RemoteConfigLongPollService 中，实现配置更新的实时通知
        this.scheduleLongPollingRefresh();
    }
```

**执行顺序**：
1. **立即同步**：`super.trySync()` - 确保启动时就有配置
2. **启动定时刷新**：`schedulePeriodicRefresh()` - 作为兜底
3. **注册长轮询**：`scheduleLongPollingRefresh()` - 实现实时推送

### 4.2 配置更新流程

```
场景1：配置变更（正常情况）
┌─────────────────┐
│ Config Service  │ 发布配置变更
└────────┬────────┘
         │ 写入 ReleaseMessage
         ↓
┌─────────────────┐
│ 长轮询请求      │ 检测到变更
│ (90秒超时)      │
└────────┬────────┘
         │ 返回 200 + 变更的 namespace
         ↓
┌─────────────────┐
│ RemoteConfig    │ onLongPollNotified()
│ Repository      │ → sync() → 拉取最新配置
└────────┬────────┘
         │ fireRepositoryChange()
         ↓
┌─────────────────┐
│ DefaultConfig   │ onRepositoryChange()
│                 │ → fireConfigChange()
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│ ConfigChange    │ onChange()
│ Listener        │ 更新 Spring Bean
└─────────────────┘

场景2：长轮询失败（异常情况）
┌─────────────────┐
│ 长轮询异常      │ 网络断开/服务端重启
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│ 定时刷新        │ 5分钟后执行
│ (兜底机制)      │ → sync() → 拉取配置
└─────────────────┘
```

### 4.3 性能优化

1. **HTTP 304 优化**：
   - 定时刷新和长轮询都使用 releaseKey
   - 配置未变更时返回 304，不传输数据

2. **优先访问策略**：
   - 长轮询通知的 Config Service 会被优先访问
   - 减少网络延迟

3. **限流保护**：
   - 定时刷新：`m_loadConfigRateLimiter`
   - 长轮询：`m_longPollRateLimiter`

4. **失败重试**：
   - 指数退避策略，避免频繁重试

---

## 五、总结

### 5.1 设计思想

Apollo 采用**双重保障 + 优化机制**的设计：

1. **长轮询为主**：实现配置的准实时推送（< 1 秒）
2. **定时刷新为辅**：作为兜底机制，确保配置最终一致性
3. **HTTP 304 优化**：减少无效数据传输
4. **限流保护**：防止服务端压力过大

### 5.2 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 定时刷新间隔 | 5 分钟 | 兜底机制 |
| 长轮询超时 | 90 秒 | 比服务端 60 秒长 |
| 配置拉取 QPS | 2 | 限流保护 |
| 长轮询 QPS | 1 | 限流保护 |

### 5.3 为什么这样设计？

1. **实时性**：长轮询保证配置变更在 1 秒内感知
2. **可靠性**：定时刷新确保即使长轮询失败，配置也能更新
3. **性能**：HTTP 304 + 限流，减少服务端压力
4. **容错性**：多 Config Service 负载均衡 + 失败重试

这种设计在**实时性、可靠性和性能**之间取得了很好的平衡。

