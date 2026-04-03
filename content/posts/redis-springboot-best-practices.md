+++
date = '2026-04-03T00:00:00+08:00'
draft = false
title = 'Redis + Spring Boot 最佳实践完整教程'
tags = ["Redis", "Spring Boot", "缓存", "消息队列", "分布式锁"]
categories = ["技术教程"]
+++

## Redis + Spring Boot 最佳实践完整教程

Redis 作为一款高性能的内存数据库，在现代 Web 应用中扮演着越来越重要的角色。Spring Boot 则提供了强大的自动配置能力，让 Redis 的集成变得异常简单。本文将带你从零开始，全面掌握 Redis + Spring Boot 的最佳实践。

## 项目搭建与基础配置

### 1. 依赖配置

首先在 `pom.xml` 中添加必要的依赖：

```xml
<!-- Spring Boot Data Redis Starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Spring Boot Cache Starter (可选，用于声明式缓存) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Lombok (可选，简化代码) -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### 2. 应用配置

在 `application.yml` 中配置 Redis 连接信息：

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      database: 0
      timeout: 3000
      lettuce:
        pool:
          max-active: 20
          max-wait: -1
          max-idle: 10
          min-idle: 5
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10分钟
      cache-null-values: false
      key-prefix: "cache:"
      use-key-prefix: true
```

### 3. RedisTemplate 配置

配置 RedisTemplate 的序列化方式是关键，推荐使用 Jackson 序列化：

```java
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // 使用 Jackson2JsonRedisSerializer 序列化 value
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, 
                                    ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);
        
        // Key 使用 String 序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Value 使用 JSON 序列化
        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
}
```

## Redis 数据结构操作

### 1. 字符串 (String)

```java
@Service
public class RedisStringService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 设置键值对
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }
    
    // 设置带过期时间的键值对
    public void set(String key, Object value, long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }
    
    // 获取值
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }
    
    // 原子递增
    public Long increment(String key) {
        return redisTemplate.opsForValue().increment(key);
    }
    
    // 如果不存在则设置
    public Boolean setIfAbsent(String key, Object value) {
        return redisTemplate.opsForValue().setIfAbsent(key, value);
    }
}
```

### 2. 列表 (List)

```java
@Service
public class RedisListService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 从左边推入
    public Long leftPush(String key, Object value) {
        return redisTemplate.opsForList().leftPush(key, value);
    }
    
    // 从右边推入
    public Long rightPush(String key, Object value) {
        return redisTemplate.opsForList().rightPush(key, value);
    }
    
    // 从左边弹出
    public Object leftPop(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }
    
    // 获取列表范围
    public List<Object> range(String key, long start, long end) {
        return redisTemplate.opsForList().range(key, start, end);
    }
}
```

### 3. 哈希 (Hash)

```java
@Service
public class RedisHashService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 设置哈希字段
    public void put(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
    
    // 获取哈希字段
    public Object get(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }
    
    // 获取所有哈希字段
    public Map<Object, Object> entries(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
}
```

### 4. 集合 (Set)

```java
@Service
public class RedisSetService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 添加元素
    public Long add(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }
    
    // 获取所有成员
    public Set<Object> members(String key) {
        return redisTemplate.opsForSet().members(key);
    }
    
    // 判断成员是否存在
    public Boolean isMember(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }
}
```

## Spring Cache 声明式缓存

### 1. 启用缓存

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new Jackson2JsonRedisSerializer<>(Object.class)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .transactionAware()
            .build();
    }
}
```

### 2. 使用缓存注解

```java
@Service
public class UserService {
    
    // 查询结果缓存
    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    public User getUserById(Long id) {
        System.out.println("从数据库查询用户: " + id);
        User user = new User();
        user.setId(id);
        user.setName("用户" + id);
        return user;
    }
    
    // 更新缓存
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        System.out.println("更新用户: " + user.getId());
        return user;
    }
    
    // 清除缓存
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        System.out.println("删除用户: " + id);
    }
}
```

## Redis 消息队列

Redis 提供了三种实现消息队列的方式，各有优劣：

### 1. List：简单队列

```java
@Component
public class ListQueueProducer {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public void send(String message) {
        redisTemplate.opsForList().leftPush("order_queue", message);
    }
}

@Component
public class ListQueueConsumer {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Scheduled(fixedDelay = 500)
    public void receive() {
        String message = redisTemplate.opsForList().rightPop("order_queue");
        if (message != null) {
            System.out.println("收到消息：" + message);
        }
    }
}
```

### 2. Pub/Sub：广播模式

```java
@Component
public class PubSubProducer {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public void send(String message) {
        redisTemplate.convertAndSend("order_channel", message);
    }
}

@Component
public class PubSubConsumer {
    
    @RedisListener(channels = "order_channel")
    public void receive(String message) {
        System.out.println("收到消息：" + message);
    }
}
```

### 3. Stream：可靠消息队列（推荐）

```java
@Component
public class StreamProducer {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public String send(Map<String, String> fields) {
        return redisTemplate.opsForStream().add("order_stream", fields);
    }
}

@Component
public class StreamConsumer {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @PostConstruct
    public void init() {
        try {
            redisTemplate.opsForStream().createGroup("order_stream", "group1", StreamOffset.fromStart());
        } catch (Exception e) {
            // 消费组已存在
        }
    }
    
    @Scheduled(fixedDelay = 500)
    public void consume() {
        List<StreamMessage<String, String>> messages = redisTemplate.opsForStream().read(
            Consumer.from("group1", "consumer1"),
            StreamReadOptions.empty().count(10),
            StreamOffset.create("order_stream", StreamOffset.FromStart.START)
        );
        
        if (messages != null && !messages.isEmpty()) {
            for (StreamMessage<String, String> message : messages) {
                Map<String, String> fields = message.getValue();
                String messageId = message.getId();
                
                System.out.println("收到消息：" + fields);
                
                // 确认消息
                redisTemplate.opsForStream().acknowledge("order_stream", "group1", messageId);
            }
        }
    }
}
```

## 分布式锁

使用 Redis 实现分布式锁是常见的应用场景：

```java
@Service
public class RedisLockService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private static final long DEFAULT_EXPIRE_TIME = 30; // 30秒
    
    /**
     * 尝试获取锁
     */
    public boolean tryLock(String lockKey, String requestId) {
        Boolean success = redisTemplate.opsForValue().setIfAbsent(
            LOCK_PREFIX + lockKey, requestId, DEFAULT_EXPIRE_TIME, TimeUnit.SECONDS
        );
        return success != null && success;
    }
    
    /**
     * 释放锁（使用 Lua 脚本保证原子性）
     */
    public boolean releaseLock(String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                       "return redis.call('del', KEYS[1]) " +
                       "else " +
                       "return 0 " +
                       "end";
        
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(script);
        redisScript.setResultType(Long.class);
        
        Long result = redisTemplate.execute(redisScript, 
            Collections.singletonList(LOCK_PREFIX + lockKey), requestId);
        
        return result != null && result == 1;
    }
}
```

## 最佳实践与注意事项

### 1. 序列化选择
- **Key**: 使用 `StringRedisSerializer`，便于调试和查看
- **Value**: 使用 `Jackson2JsonRedisSerializer`，支持复杂对象

### 2. 连接池配置
- 根据实际并发量调整连接池大小
- 生产环境务必使用连接池，避免频繁创建连接

### 3. 缓存策略
- 合理设置过期时间，避免缓存雪崩
- 使用缓存预热提高系统性能
- 注意缓存穿透和缓存击穿问题

### 4. 性能优化
- 使用 Pipeline 批量操作，减少网络往返
- 合理使用批量操作 API
- 避免大 Key 和热 Key 问题

### 5. 监控与运维
- 监控 Redis 内存使用情况
- 定期清理过期 Key
- 使用 Redis Sentinel 或 Cluster 保证高可用

### 6. 消息队列选型建议

| 场景 | 推荐方案 | 说明 |
|------|----------|------|
| 简单异步任务 | List | 实现简单，但有消息丢失风险 |
| 实时广播 | Pub/Sub | 不支持持久化，适合对可靠性要求不高的场景 |
| 可靠消息、复杂业务 | **Stream** | 支持持久化、ACK、消费组，生产环境首选 |

## 总结

Spring Boot + Redis 的整合是现代后端开发的必备技能。通过本文的学习，你应该掌握了：

1. **基础配置**: 依赖、连接、序列化配置
2. **数据结构**: String、List、Hash、Set 的使用
3. **声明式缓存**: Spring Cache 注解的应用
4. **消息队列**: 三种实现方式及选型建议
5. **分布式锁**: Redis 分布式锁的实现
6. **最佳实践**: 生产环境的注意事项

Redis 功能强大且易于使用，配合 Spring Boot 的自动配置，可以大大提高开发效率。希望这篇教程能帮助你在项目中更好地应用 Redis！

[^1]: [Spring Boot 4 整合 Redis 完整教程](https://www.ddkk.com/springboot/4-action/10.html)
[^2]: [Redis 也能做消息队列！Spring Boot 实战：从 List 到 Stream 的优雅实现](https://juejin.cn/post/7621470473189097515)
