# Apollo 配置中心架构与实现原理分析

## 一、整体架构概览

Apollo 是一个分布式配置中心，采用**客户端-服务端**架构模式，主要包含以下核心模块：

### 1.1 模块划分

```
apollo/
├── apollo-client/          # 客户端SDK（核心）
├── apollo-configservice/   # 配置服务（提供配置查询）
├── apollo-adminservice/    # 管理服务（配置管理）
├── apollo-portal/          # 管理界面（Web UI）
├── apollo-common/          # 公共组件
├── apollo-core/            # 核心组件
└── apollo-biz/             # 业务逻辑层
```

### 1.2 架构设计思想

1. **分层架构**：客户端、服务端、数据层清晰分离
2. **观察者模式**：配置变更通过监听器机制通知
3. **工厂模式**：ConfigFactory 负责创建不同类型的 Config 实例
4. **策略模式**：支持多种配置源（远程、本地、系统属性等）
5. **缓存机制**：多级缓存提升性能（内存缓存 + 本地文件缓存）

---

## 二、核心组件分析

### 2.1 客户端核心类图

```
ConfigService (入口)
    ↓
ConfigManager (配置管理器)
    ↓
ConfigFactory (工厂) → DefaultConfigFactory
    ↓
DefaultConfig (配置实现)
    ↓
ConfigRepository (配置仓库) → RemoteConfigRepository
    ↓
RemoteConfigLongPollService (长轮询服务)
```

### 2.2 关键组件说明

#### 2.2.1 ConfigService（配置服务入口）

```java
// 单例模式，作为客户端使用配置的统一入口
public class ConfigService {
    private static final ConfigService s_instance = new ConfigService();
    private volatile ConfigManager m_configManager;
    
    public static Config getConfig(String namespace) {
        return s_instance.getManager().getConfig(namespace);
    }
}
```

**职责**：
- 提供静态方法获取 Config 实例
- 管理 ConfigManager 和 ConfigRegistry
- 单例模式确保全局唯一入口

#### 2.2.2 ConfigManager（配置管理器）

```java
public class DefaultConfigManager implements ConfigManager {
    // Config 对象缓存（按 namespace 缓存）
    private Map<String, Config> m_configs = Maps.newConcurrentMap();
    
    public Config getConfig(String namespace) {
        Config config = m_configs.get(namespace);
        if (config == null) {
            synchronized (this) {
                config = m_configs.get(namespace);
                if (config == null) {
                    ConfigFactory factory = m_factoryManager.getFactory(namespace);
                    config = factory.create(namespace);
                    m_configs.put(namespace, config);
                }
            }
        }
        return config;
    }
}
```

**职责**：
- 管理所有 Config 实例的缓存
- 延迟加载：首次访问时创建
- 线程安全：使用 ConcurrentMap + 双重检查锁定

#### 2.2.3 ConfigRepository（配置仓库）

**层次结构**：
```
ConfigRepository (接口)
    ├── AbstractConfigRepository (抽象类)
    │   ├── LocalFileConfigRepository (本地文件)
    │   └── RemoteConfigRepository (远程配置) ⭐
```

**RemoteConfigRepository 核心逻辑**：

```java
public class RemoteConfigRepository extends AbstractConfigRepository {
    // 配置缓存
    private AtomicReference<ApolloConfig> m_configCache;
    
    // 初始化流程
    public RemoteConfigRepository(String namespace) {
        // 1. 尝试同步配置
        super.trySync();
        // 2. 初始化定时刷新配置的任务
        this.schedulePeriodicRefresh();
        // 3. 注册到长轮询服务，实现配置更新的实时通知
        this.scheduleLongPollingRefresh();
    }
    
    // 同步配置
    protected synchronized void sync() {
        ApolloConfig previous = m_configCache.get();
        ApolloConfig current = loadApolloConfig();
        
        if (previous != current) {
            m_configCache.set(current);
            // 发布配置变更事件
            super.fireRepositoryChange(m_namespace, this.getConfig());
        }
    }
}
```

**职责**：
- 从 Config Service 拉取配置
- 管理配置缓存
- 触发配置变更通知

---

## 三、配置加载流程

### 3.1 启动流程（基于 Spring Boot）

#### 阶段1：Bootstrap 阶段（ApplicationContextInitializer）

```java
// ApolloApplicationContextInitializer
public void initialize(ConfigurableApplicationContext context) {
    // 1. 检查是否启用 Apollo
    String enabled = environment.getProperty("apollo.bootstrap.enabled", "false");
    if (!Boolean.valueOf(enabled)) {
        return;
    }
    
    // 2. 获取要加载的 namespaces
    String namespaces = environment.getProperty("apollo.bootstrap.namespaces", "application");
    
    // 3. 为每个 namespace 创建 Config 并添加到 PropertySource
    CompositePropertySource composite = new CompositePropertySource("ApolloBootstrapPropertySources");
    for (String namespace : namespaceList) {
        Config config = ConfigService.getConfig(namespace);
        composite.addPropertySource(
            configPropertySourceFactory.getConfigPropertySource(namespace, config)
        );
    }
    
    // 4. 添加到 Environment 的最前面（最高优先级）
    environment.getPropertySources().addFirst(composite);
}
```

**关键点**：
- 在 Spring Boot 启动的**最早阶段**注入配置
- 确保 Apollo 配置优先于其他配置源
- 通过 `META-INF/spring.factories` 自动注册

#### 阶段2：Bean 初始化阶段（PropertySourcesProcessor）

```java
// PropertySourcesProcessor (BeanFactoryPostProcessor)
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 1. 初始化 PropertySource
    initializePropertySources();
    // 2. 初始化自动更新功能
    initializeAutoUpdatePropertiesFeature(beanFactory);
}

private void initializePropertySources() {
    CompositePropertySource composite = new CompositePropertySource("ApolloPropertySources");
    // 按优先级顺序添加 namespaces
    for (Integer order : orders) {
        for (String namespace : NAMESPACE_NAMES.get(order)) {
            Config config = ConfigService.getConfig(namespace);
            composite.addPropertySource(
                configPropertySourceFactory.getConfigPropertySource(namespace, config)
            );
        }
    }
    environment.getPropertySources().addAfter(
        PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME, 
        composite
    );
}
```

**关键点**：
- 支持多个 namespace，按 order 排序
- 通过 `@EnableApolloConfig` 注解注册

### 3.2 配置获取流程

```
用户调用 ConfigService.getConfig(namespace)
    ↓
ConfigManager.getConfig(namespace)
    ↓
检查缓存 → 缓存未命中
    ↓
ConfigFactory.create(namespace)
    ↓
DefaultConfigFactory.create()
    ↓
创建 DefaultConfig + RemoteConfigRepository
    ↓
RemoteConfigRepository 初始化：
    1. 从 Config Service 拉取配置（HTTP GET）
    2. 启动定时刷新任务
    3. 启动长轮询任务
    ↓
返回 Config 实例
```

### 3.3 配置优先级

在 `DefaultConfig.getProperty()` 方法中，配置获取的优先级顺序：

```java
public String getProperty(String key, String defaultValue) {
    // 1. 系统属性（最高优先级）-Dkey=value
    String value = System.getProperty(key);
    
    // 2. Apollo 远程配置缓存
    if (value == null && m_configProperties.get() != null) {
        value = m_configProperties.get().getProperty(key);
    }
    
    // 3. 环境变量
    if (value == null) {
        value = System.getenv(key);
    }
    
    // 4. 本地资源文件（classpath:META-INF/config/{namespace}.properties）
    if (value == null && m_resourceProperties != null) {
        value = (String) m_resourceProperties.get(key);
    }
    
    // 5. 默认值
    return value == null ? defaultValue : value;
}
```

**优先级顺序**：系统属性 > Apollo远程配置 > 环境变量 > 本地资源文件 > 默认值

---

## 四、配置变更通知机制

### 4.1 长轮询（Long Polling）机制

Apollo 使用**长轮询**实现配置的实时推送，避免频繁轮询。

#### 4.1.1 长轮询流程

```java
// RemoteConfigLongPollService
private void doLongPollingRefresh(String appId, String cluster, String dataCenter) {
    while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
        // 1. 限流控制
        if (!m_longPollRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
            TimeUnit.SECONDS.sleep(5);
        }
        
        // 2. 组装长轮询 URL
        String url = assembleLongPollRefreshUrl(
            lastServiceDto.getHomepageUrl(), 
            appId, cluster, dataCenter, 
            m_notifications  // 携带当前所有 namespace 的 notificationId
        );
        
        // 3. 发起长轮询请求（超时时间 90 秒）
        HttpRequest request = new HttpRequest(url);
        request.setReadTimeout(LONG_POLLING_READ_TIMEOUT); // 90秒
        
        HttpResponse<List<ApolloConfigNotification>> response = 
            m_httpUtil.doGet(request, m_responseType);
        
        // 4. 处理响应
        if (response.getStatusCode() == 200 && response.getBody() != null) {
            // 有配置变更通知
            updateNotifications(response.getBody());
            updateRemoteNotifications(response.getBody());
            // 通知对应的 RemoteConfigRepository
            notify(lastServiceDto, response.getBody());
        } else if (response.getStatusCode() == 304) {
            // 无变更，继续长轮询
            // 随机切换 Config Service 实现负载均衡
            if (random.nextBoolean()) {
                lastServiceDto = null;
            }
        }
    }
}
```

**关键点**：
- 超时时间 90 秒（比服务端 60 秒长）
- 携带所有 namespace 的 notificationId
- 304 表示无变更，继续轮询
- 200 表示有变更，立即处理

#### 4.1.2 配置变更通知链

```
Config Service 检测到配置变更
    ↓
写入 ReleaseMessage 表
    ↓
长轮询请求返回 200（携带变更的 namespace）
    ↓
RemoteConfigLongPollService.notify()
    ↓
RemoteConfigRepository.onLongPollNotified()
    ↓
RemoteConfigRepository.sync() 重新拉取配置
    ↓
检测到配置变化 → fireRepositoryChange()
    ↓
DefaultConfig.onRepositoryChange()
    ↓
计算配置变更差异 → fireConfigChange()
    ↓
通知所有 ConfigChangeListener
    ↓
AutoUpdateConfigChangeListener（自动更新 Spring Bean 属性）
```

### 4.2 配置变更监听器

#### 4.2.1 监听器接口

```java
public interface ConfigChangeListener {
    void onChange(ConfigChangeEvent changeEvent);
}
```

#### 4.2.2 自动更新 Spring Bean 属性

```java
// AutoUpdateConfigChangeListener
public void onChange(ConfigChangeEvent changeEvent) {
    Set<String> changedKeys = changeEvent.changedKeys();
    
    for (String key : changedKeys) {
        // 1. 查找所有使用该 key 的 SpringValue
        Collection<SpringValue> targetValues = 
            springValueRegistry.get(key);
        
        // 2. 更新每个 SpringValue 对应的 Bean 属性
        for (SpringValue val : targetValues) {
            updateSpringValue(val, changeEvent);
        }
    }
}
```

**支持的注解**：
- `@Value("${key}")` - 自动更新字段/方法参数
- `@ApolloJsonValue("${key}")` - 自动更新 JSON 字段
- `@ConfigurationProperties` + `@RefreshScope` - 自动刷新配置类

---

## 五、Spring Boot 集成机制

### 5.1 自动配置

#### 5.1.1 Spring Factories 机制

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.ctrip.framework.apollo.spring.boot.ApolloAutoConfiguration

org.springframework.context.ApplicationContextInitializer=\
  com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer
```

#### 5.1.2 注解驱动

```java
// @EnableApolloConfig 注解
@Import(ApolloConfigRegistrar.class)
public @interface EnableApolloConfig {
    String[] value() default {ConfigConsts.NAMESPACE_APPLICATION};
    int order() default Ordered.LOWEST_PRECEDENCE;
}

// ApolloConfigRegistrar 注册 Bean
public class ApolloConfigRegistrar implements ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(...) {
        // 1. 注册 PropertySourcesProcessor
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(
            registry, PropertySourcesProcessor.class.getName(), 
            PropertySourcesProcessor.class
        );
        
        // 2. 注册 ApolloAnnotationProcessor（处理 @ApolloConfig）
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(
            registry, ApolloAnnotationProcessor.class.getName(),
            ApolloAnnotationProcessor.class
        );
        
        // 3. 注册 SpringValueProcessor（处理 @Value）
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(
            registry, SpringValueProcessor.class.getName(), 
            SpringValueProcessor.class
        );
    }
}
```

### 5.2 Bean 后处理器

#### 5.2.1 ApolloAnnotationProcessor

处理 `@ApolloConfig` 注解，注入 Config 实例：

```java
public class ApolloAnnotationProcessor extends ApolloProcessor {
    protected void processField(Object bean, String beanName, Field field) {
        ApolloConfig annotation = AnnotationUtils.getAnnotation(field, ApolloConfig.class);
        if (annotation == null) {
            return;
        }
        
        String namespace = annotation.value();
        Config config = ConfigService.getConfig(namespace);
        
        ReflectionUtils.makeAccessible(field);
        ReflectionUtils.setField(field, bean, config);
    }
}
```

#### 5.2.2 SpringValueProcessor

处理 `@Value` 注解，注册 SpringValue 用于自动更新：

```java
public class SpringValueProcessor extends ApolloProcessor {
    protected void processField(Object bean, String beanName, Field field) {
        Value annotation = AnnotationUtils.getAnnotation(field, Value.class);
        if (annotation == null) {
            return;
        }
        
        String placeholder = annotation.value();
        // 解析占位符，提取配置 key
        Set<String> keys = placeholderResolver.extractPlaceholderKeys(placeholder);
        
        // 创建 SpringValue 并注册
        SpringValue springValue = new SpringValue(key, placeholder, bean, beanName, field, null);
        springValueRegistry.register(key, springValue);
    }
}
```

### 5.3 示例代码分析

基于启动类 `SpringBootSampleApplication` 的配置使用：

```java
@SpringBootApplication(scanBasePackages = {...})
public class SpringBootSampleApplication {
    public static void main(String[] args) {
        ApplicationContext context = new SpringApplicationBuilder(...).run(args);
        
        // 1. 获取使用 @Value 注解的 Bean
        AnnotatedBean annotatedBean = context.getBean(AnnotatedBean.class);
        
        // 2. 获取使用 @ConfigurationProperties 的 Bean（条件加载）
        SampleRedisConfig redisConfig = null;
        try {
            redisConfig = context.getBean(SampleRedisConfig.class);
        } catch (NoSuchBeanDefinitionException ex) {
            // 如果 redis.cache.enabled=false，该 Bean 不会被创建
        }
    }
}
```

**配置类示例**：

```java
// SampleRedisConfig - 使用 @ConfigurationProperties + @RefreshScope
@ConditionalOnProperty("redis.cache.enabled")
@ConfigurationProperties(prefix = "redis.cache")
@Component
@RefreshScope  // 支持配置热更新
public class SampleRedisConfig {
    private int expireSeconds;
    private String clusterNodes;
    // ...
}

// AnnotatedBean - 使用 @Value
@Component
public class AnnotatedBean {
    @Value("${timeout:200}")  // 支持默认值
    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }
    
    @ApolloJsonValue("${jsonBeanProperty:[]}")  // JSON 自动解析
    private List<JsonBean> jsonBeans;
}
```

---

## 六、服务端架构

### 6.1 Config Service（配置服务）

**职责**：
- 提供配置查询接口（HTTP GET）
- 提供长轮询接口（notifications/v2）
- 从数据库读取配置并缓存

**关键接口**：
```
GET /configs/{appId}/{clusterName}/{namespaceName}
  - 返回配置内容
  - 支持 304 Not Modified（基于 releaseKey）

GET /notifications/v2
  - 长轮询接口
  - 返回有变更的 namespace 列表
```

### 6.2 Admin Service（管理服务）

**职责**：
- 配置的增删改查
- 配置发布
- 配置回滚

### 6.3 Portal（管理界面）

**职责**：
- Web UI 管理配置
- 权限管理
- 配置审计

### 6.4 数据库设计

**核心表**：
- `App` - 应用信息
- `Namespace` - 命名空间
- `Item` - 配置项（Key-Value）
- `Release` - 发布版本
- `ReleaseMessage` - 发布消息（用于长轮询）

**配置发布流程**：
```
1. 在 Portal 修改配置 → 写入 Item 表
2. 发布配置 → 创建 Release 记录
3. 写入 ReleaseMessage 表（触发长轮询）
4. Config Service 读取 Release 并缓存
5. 客户端通过长轮询获取变更通知
```

---

## 七、关键设计模式

### 7.1 工厂模式

```java
// ConfigFactory 接口
public interface ConfigFactory {
    Config create(String namespace);
    ConfigFile createConfigFile(String namespace, ConfigFileFormat format);
}

// DefaultConfigFactory 实现
public class DefaultConfigFactory implements ConfigFactory {
    public Config create(String namespace) {
        // 创建 ConfigRepository
        ConfigRepository configRepository = createConfigRepository(namespace);
        // 创建 DefaultConfig
        return new DefaultConfig(namespace, configRepository);
    }
}
```

### 7.2 观察者模式

```java
// 监听器接口
public interface ConfigChangeListener {
    void onChange(ConfigChangeEvent changeEvent);
}

// 注册监听器
config.addChangeListener(new ConfigChangeListener() {
    public void onChange(ConfigChangeEvent changeEvent) {
        // 处理配置变更
    }
});
```

### 7.3 策略模式

配置获取的优先级策略（系统属性 > 远程配置 > 环境变量 > 本地文件）

### 7.4 单例模式

- `ConfigService` - 全局唯一入口
- `RemoteConfigLongPollService` - 全局唯一长轮询服务

---

## 八、性能优化

### 8.1 多级缓存

1. **内存缓存**：`AtomicReference<ApolloConfig>` 存储配置
2. **本地文件缓存**：`LocalFileConfigRepository` 持久化到本地
3. **HTTP 304**：基于 releaseKey 判断配置是否变更

### 8.2 限流机制

```java
// 配置拉取限流
private RateLimiter m_loadConfigRateLimiter = 
    RateLimiter.create(m_configUtil.getLoadConfigQPS());

// 长轮询限流
private RateLimiter m_longPollRateLimiter = 
    RateLimiter.create(m_configUtil.getLongPollQPS());
```

### 8.3 重试策略

```java
// 指数退避重试
private SchedulePolicy m_loadConfigFailSchedulePolicy = 
    new ExponentialSchedulePolicy(
        m_configUtil.getOnErrorRetryInterval(), 
        m_configUtil.getOnErrorRetryInterval() * 8
    );
```

---

## 九、学习建议

### 9.1 阅读顺序

1. **入口类**：`ConfigService` → `ConfigManager` → `DefaultConfig`
2. **配置加载**：`RemoteConfigRepository` → `ConfigServiceLocator`
3. **变更通知**：`RemoteConfigLongPollService` → `ConfigChangeListener`
4. **Spring 集成**：`ApolloApplicationContextInitializer` → `PropertySourcesProcessor` → `ApolloAnnotationProcessor`

### 9.2 关键文件清单

**客户端核心**：
- `apollo-client/src/main/java/com/ctrip/framework/apollo/ConfigService.java`
- `apollo-client/src/main/java/com/ctrip/framework/apollo/internals/DefaultConfig.java`
- `apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigRepository.java`
- `apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java`

**Spring 集成**：
- `apollo-client/src/main/java/com/ctrip/framework/apollo/spring/boot/ApolloApplicationContextInitializer.java`
- `apollo-client/src/main/java/com/ctrip/framework/apollo/spring/config/PropertySourcesProcessor.java`
- `apollo-client/src/main/java/com/ctrip/framework/apollo/spring/annotation/ApolloAnnotationProcessor.java`

### 9.3 调试技巧

1. **开启日志**：设置日志级别为 DEBUG
2. **断点位置**：
   - `RemoteConfigRepository.sync()` - 配置同步
   - `RemoteConfigLongPollService.doLongPollingRefresh()` - 长轮询
   - `DefaultConfig.onRepositoryChange()` - 配置变更处理
3. **配置验证**：使用启动类示例，观察配置加载和更新过程

---

## 十、总结

Apollo 的核心设计思想：

1. **客户端缓存 + 长轮询**：保证配置实时性的同时减少服务端压力
2. **多级配置源**：系统属性 > 远程配置 > 环境变量 > 本地文件，提供灵活的配置覆盖机制
3. **Spring 深度集成**：通过 ApplicationContextInitializer 和 BeanPostProcessor 实现无缝集成
4. **配置热更新**：通过监听器机制自动更新 Spring Bean 属性，无需重启应用
5. **高可用设计**：支持多 Config Service 负载均衡，失败重试机制

通过理解这些核心机制，可以更好地使用和扩展 Apollo 配置中心。

