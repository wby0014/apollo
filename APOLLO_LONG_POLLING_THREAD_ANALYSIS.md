# Apollo 长轮询任务持续执行与线程中断处理分析

## 一、问题分析

### 1.1 代码结构

```159:175:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
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
```

**关键问题**：
1. 只提交了一次任务到线程池，如何保证持续循环执行？
2. 如果线程被中断，如何处理？

---

## 二、持续循环执行机制

### 2.1 循环在方法内部实现

**关键点**：循环不是在提交的 Runnable 外部，而是在 `doLongPollingRefresh()` 方法**内部**实现的！

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

**执行流程**：

```
提交任务到线程池
    ↓
Runnable.run() 执行
    ↓
调用 doLongPollingRefresh()
    ↓
进入 while 循环
    ↓
执行一次长轮询请求
    ↓
处理响应（200/304/异常）
    ↓
循环回到 while 条件检查
    ↓
如果条件满足，继续循环
    ↓
如果条件不满足，退出循环，任务结束
```

### 2.2 循环条件

```java
while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted())
```

**两个退出条件**：
1. `m_longPollingStopped.get() == true`：通过 `stopLongPollingRefresh()` 方法设置
2. `Thread.currentThread().isInterrupted() == true`：线程被中断

### 2.3 为什么只提交一次任务？

**设计原因**：

1. **单线程执行**：
   ```java
   m_longPollingService = Executors.newSingleThreadExecutor(
       ApolloThreadFactory.create("RemoteConfigLongPollService", true)
   );
   ```
   - 使用 `SingleThreadExecutor`，确保只有一个线程执行长轮询
   - 避免并发问题

2. **持续循环**：
   - 在 `doLongPollingRefresh()` 方法内部使用 `while` 循环
   - 只要条件满足，就会一直循环执行
   - **不需要重复提交任务**

3. **资源效率**：
   - 只占用一个线程
   - 避免频繁创建和销毁线程的开销

---

## 三、线程中断处理机制

### 3.1 中断检查位置

#### 3.1.1 循环条件检查

```java
while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
    // 循环体
}
```

**作用**：每次循环开始前检查线程是否被中断，如果中断则退出循环。

#### 3.1.2 Sleep 中断处理

在代码中有**三处** `sleep` 操作，都捕获了 `InterruptedException`：

**位置1：初始延迟**
```163:170:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
                    // 初始等待
                    if (longPollingInitialDelayInMills > 0) {
                        try {
                            logger.debug("Long polling will start in {} ms.", longPollingInitialDelayInMills);
                            TimeUnit.MILLISECONDS.sleep(longPollingInitialDelayInMills);
                        } catch (InterruptedException e) {
                            //ignore
                        }
                    }
```

**位置2：限流等待**
```205:209:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
                try {
                    TimeUnit.SECONDS.sleep(5);
                } catch (InterruptedException e) {
                }
```

**位置3：失败重试等待**
```268:272:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
                try {
                    TimeUnit.SECONDS.sleep(sleepTimeInSecond);
                } catch (InterruptedException ie) {
                    //ignore
                }
```

### 3.2 中断处理的问题

#### 3.2.1 问题：中断标志被清除

**Java 中断机制**：
- 当线程在 `sleep()`、`wait()`、`join()` 等阻塞操作中被中断时，会抛出 `InterruptedException`
- **抛出异常后，中断标志会被清除**（`isInterrupted()` 返回 `false`）

**当前代码的问题**：
```java
try {
    TimeUnit.SECONDS.sleep(5);
} catch (InterruptedException e) {
    //ignore  ← 只是忽略，没有重新设置中断标志
}
```

**后果**：
- 如果 `sleep()` 被中断，中断标志被清除
- 下次循环时，`Thread.currentThread().isInterrupted()` 返回 `false`
- **循环可能不会立即退出**，需要等到下一次循环或下一次 sleep 被中断

#### 3.2.2 为什么代码这样写？

**可能的原因**：

1. **设计意图**：
   - 允许在 sleep 期间被中断，但不强制退出
   - 继续执行长轮询逻辑，而不是立即停止

2. **实际效果**：
   - 如果 sleep 被中断，会立即继续执行
   - 如果循环条件检查时线程已中断，会退出循环

3. **潜在风险**：
   - 如果中断发生在 sleep 中，但循环条件检查时中断标志已被清除，可能不会立即退出

### 3.3 改进建议

#### 3.3.1 标准的中断处理方式

**方式1：重新设置中断标志**

```java
try {
    TimeUnit.SECONDS.sleep(5);
} catch (InterruptedException e) {
    // 重新设置中断标志
    Thread.currentThread().interrupt();
    // 可以选择立即退出或继续执行
    break; // 或者 return
}
```

**方式2：使用标志变量**

```java
private volatile boolean shouldStop = false;

// 在 sleep 中
try {
    TimeUnit.SECONDS.sleep(5);
} catch (InterruptedException e) {
    shouldStop = true; // 设置停止标志
    Thread.currentThread().interrupt(); // 恢复中断标志
}

// 在循环条件中
while (!m_longPollingStopped.get() && !shouldStop && !Thread.currentThread().isInterrupted()) {
    // ...
}
```

#### 3.3.2 Apollo 当前实现的合理性

**为什么 Apollo 这样实现可能是合理的**：

1. **长轮询的特殊性**：
   - 长轮询请求本身有 90 秒超时
   - 即使 sleep 被中断，也不会影响长轮询请求的执行
   - 循环条件检查会在下一次迭代时捕获中断

2. **优雅退出**：
   - 使用 `m_longPollingStopped` 标志进行优雅停止
   - 不依赖线程中断作为主要停止机制

3. **实际场景**：
   - 在应用关闭时，通常会调用 `stopLongPollingRefresh()` 设置标志
   - 线程中断可能用于强制停止，但不是主要方式

---

## 四、完整的执行流程

### 4.1 正常执行流程

```
1. startLongPolling() 被调用
   ↓
2. 提交 Runnable 到 SingleThreadExecutor
   ↓
3. 线程执行 Runnable.run()
   ↓
4. 执行初始延迟（如果有）
   ↓
5. 调用 doLongPollingRefresh()
   ↓
6. 进入 while 循环
   ↓
7. 执行长轮询请求（90秒超时）
   ↓
8. 处理响应（200/304/异常）
   ↓
9. 检查循环条件
   ↓
10. 如果条件满足，回到步骤 6（继续循环）
   如果条件不满足，退出循环，任务结束
```

### 4.2 中断处理流程

**场景1：在循环条件检查时中断**

```
1. 线程被中断（interrupt()）
   ↓
2. 下一次循环条件检查
   ↓
3. Thread.currentThread().isInterrupted() 返回 true
   ↓
4. 循环条件为 false，退出循环
   ↓
5. 任务结束
```

**场景2：在 sleep 时中断**

```
1. 线程在 sleep(5) 中被中断
   ↓
2. 抛出 InterruptedException
   ↓
3. 捕获异常，忽略（不重新设置中断标志）
   ↓
4. 继续执行后续代码
   ↓
5. 下一次循环条件检查
   ↓
6. 如果中断标志已被清除，循环继续
   如果中断标志仍存在，循环退出
```

**场景3：在 HTTP 请求时中断**

```
1. 线程在 HTTP 请求（90秒超时）中被中断
   ↓
2. HTTP 客户端可能抛出异常或正常返回
   ↓
3. 处理响应或异常
   ↓
4. 下一次循环条件检查
   ↓
5. 如果中断标志存在，循环退出
```

### 4.3 停止机制

#### 4.3.1 优雅停止

```186:188:apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java
    void stopLongPollingRefresh() {
        this.m_longPollingStopped.compareAndSet(false, true);
    }
```

**流程**：
1. 调用 `stopLongPollingRefresh()` 设置 `m_longPollingStopped = true`
2. 下一次循环条件检查时，`!m_longPollingStopped.get()` 返回 `false`
3. 循环退出，任务结束

#### 4.3.2 强制停止

**方式1：中断线程**
```java
// 获取线程并中断
Thread longPollingThread = // ... 获取线程
longPollingThread.interrupt();
```

**方式2：关闭线程池**
```java
m_longPollingService.shutdown(); // 优雅关闭
// 或
m_longPollingService.shutdownNow(); // 强制关闭（会中断所有任务）
```

---

## 五、潜在问题与改进

### 5.1 当前实现的问题

1. **中断标志丢失**：
   - 在 sleep 中捕获 `InterruptedException` 后，没有重新设置中断标志
   - 可能导致循环不能立即退出

2. **HTTP 请求中断处理不明确**：
   - 如果 HTTP 请求正在进行时线程被中断，行为不明确
   - 取决于 HTTP 客户端的实现

3. **缺少中断日志**：
   - 中断被静默处理，没有日志记录
   - 难以排查问题

### 5.2 改进建议

#### 改进1：正确处理中断

```java
private void doLongPollingRefresh(String appId, String cluster, String dataCenter) {
    final Random random = new Random();
    ServiceDTO lastServiceDto = null;
    
    while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
        // 限流
        if (!m_longPollRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                // 记录中断日志
                logger.info("Long polling interrupted during rate limiter wait");
                // 重新设置中断标志
                Thread.currentThread().interrupt();
                // 退出循环
                break;
            }
        }
        
        // ... 长轮询逻辑 ...
        
        // 在异常处理中
        catch (Throwable ex) {
            // ... 错误处理 ...
            try {
                TimeUnit.SECONDS.sleep(sleepTimeInSecond);
            } catch (InterruptedException ie) {
                logger.info("Long polling interrupted during retry wait");
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    // 退出循环后的清理工作
    logger.info("Long polling stopped. Reason: stopped={}, interrupted={}", 
        m_longPollingStopped.get(), Thread.currentThread().isInterrupted());
}
```

#### 改进2：使用标志变量

```java
private volatile boolean interrupted = false;

private void doLongPollingRefresh(...) {
    while (!m_longPollingStopped.get() && !interrupted && !Thread.currentThread().isInterrupted()) {
        try {
            // ... 业务逻辑 ...
        } catch (InterruptedException e) {
            interrupted = true;
            Thread.currentThread().interrupt();
            logger.info("Long polling interrupted");
            break;
        }
    }
}
```

---

## 六、总结

### 6.1 持续循环机制

1. **循环在方法内部**：`doLongPollingRefresh()` 方法内部使用 `while` 循环
2. **只提交一次任务**：任务提交到线程池后，通过内部循环持续执行
3. **单线程执行**：使用 `SingleThreadExecutor`，确保只有一个线程执行

### 6.2 线程中断处理

1. **循环条件检查**：每次循环开始前检查 `Thread.currentThread().isInterrupted()`
2. **Sleep 中断捕获**：三处 sleep 都捕获了 `InterruptedException`，但只是忽略
3. **潜在问题**：中断标志可能被清除，导致循环不能立即退出

### 6.3 停止机制

1. **优雅停止**：通过 `stopLongPollingRefresh()` 设置标志
2. **强制停止**：通过线程中断或关闭线程池

### 6.4 设计权衡

Apollo 的当前实现：
- ✅ **优点**：简单，适合长轮询场景
- ⚠️ **缺点**：中断处理不够完善，可能在某些场景下不能立即响应中断

**建议**：
- 如果对中断响应要求不高，当前实现可以接受
- 如果需要更可靠的中断处理，建议改进中断标志的处理方式

