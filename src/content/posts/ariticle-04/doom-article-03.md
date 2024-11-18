---
title: 使用 Redis 和 Caffeine 构建多级缓存
published: 2024-11-18
description: ''
image: './pic.jpg'
tags: [Java, Redis, Caffeine]
category: 'Cache'
draft: false 
lang: 'zh_CN'
---

# 使用 Redis 和 Caffeine 构建多级缓存
在分布式系统中，缓存是提升性能、降低数据库压力的关键组件。通过缓存，我们可以快速响应用户请求，避免对数据库的频繁访问。本文将介绍如何使用 Redis 和 Caffeine 构建多级缓存，探索其原理和实现。

## 一、什么是 [Redis][Redis] 和 [Caffeine][Caffeine]？
### 1. Redis
[Redis][Redis] 是一个基于内存的高性能分布式缓存系统。它支持多种数据结构（如字符串、哈希、列表、集合等），拥有强大的持久化和分布式特性。Redis 常用来作为一级缓存，也可以用于存储全局会话、限流等场景。

### 特点
- 数据存储在内存中，读写速度极快。
- 分布式支持，能够扩展为高可用集群。
- 支持丰富的数据结构和 Lua 脚本扩展。

### 2.Caffeine
[Caffeine][Caffeine] 是 Java 平台上一款高效的本地缓存库，由 Google Guava Cache 的作者开发。它专为高性能设计，常用于应用的本地缓存层。Caffeine 提供丰富的特性，如 LRU、LFU 淘汰策略和异步加载。

### 特点
- 本地缓存（存储在 JVM 内存中），访问延迟极低。
- 支持灵活的缓存回收策略（基于时间、大小或自定义规则）。
- 高并发访问，性能优越。

## 二、为什么要构建多级缓存？
单一的缓存层通常不能满足高性能、高可用的业务需求。例如：
- **仅用 Caffeine**： 本地缓存虽然访问速度快，但内存有限，缓存容量受限，且无法在分布式环境中共享数据。
- **仅用 Redis**： Redis 作为远程缓存，存在网络传输延迟，无法与本地缓存的速度媲美。

**多级缓存**通过结合本地缓存（Caffeine）和分布式缓存（Redis）的优点，解决了这些问题。

### 优势
**1. 提升性能**： 本地缓存优先命中，减少远程缓存和数据库访问。
**2. 减轻 Redis 压力**： 避免频繁访问 Redis，大幅降低 Redis 负载。
**3. 保证数据一致性**： 使用旁路缓存策略等机制，保证缓存与数据库一致性。

## 三、多级缓存策略设计
**旁路缓存策略**（Cache-Aside Pattern）是多级缓存的经典模式，主要思路如下：

**1.查询数据**：
- 先查询 Caffeine（本地缓存）；
- 如果未命中，则查询 Redis（远程缓存）；
- 如果仍未命中，则查询数据库，并将结果分别写入 Redis 和 Caffeine。

**2.更新数据**：
- 数据库更新后，清理 Redis 和 Caffeine 中的缓存数据。

**3.删除数据**：
- 数据删除时，移除 Redis 和 Caffeine 中的相关缓存数据。

## 四、实现过程

### 1. 使用 Caffeine 实现单级缓存
我们先通过 Caffeine 在 Spring Boot 中实现单级缓存，并通过注解方式进行简单的增删改查缓存管理。

#### （1）引入依赖
在 `pom.xml` 文件中添加以下依赖：

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

[spring-boot-starter-cache](https://docs.spring.io/spring-boot/reference/io/caching.html)提供了自动配置和一些默认的缓存管理器实现。

#### （2）配置 Caffeine 缓存
在 `application.yml` 中添加缓存配置：

```yaml
spring:
  cache:
    type: caffeine
  caffeine:
    spec: maximumSize=1000,expireAfterWrite=5m
```

#### （3）开启缓存注解
在主程序类中添加 `@EnableCaching` 注解：

```java
@SpringBootApplication
@EnableCaching
public class CacheApplication {
    public static void main(String[] args) {
        SpringApplication.run(CacheApplication.class, args);
    }
}
```

#### （4）实现业务逻辑
假设我们有一个用户管理的服务类，包含查询、更新和删除操作：

```java
@Service
public class UserService {

    private final Map<Long, User> database = new HashMap<>();

    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        // 模拟从数据库查询
        System.out.println("Fetching user from database...");
        return database.get(id);
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        // 模拟更新数据库
        System.out.println("Updating user in database...");
        database.put(user.getId(), user);
        return user;
    }

    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        // 模拟删除数据库记录
        System.out.println("Deleting user from database...");
        database.remove(id);
    }
}
```

### 2. 使用 Redis + Caffeine 实现多级缓存
#### （1）引入依赖
在 `pom.xml` 文件中添加 Redis 和 Caffeine 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!--common-pool-->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
<!--jackson-->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>

<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

#### （2）配置 Redis 和 Caffeine
在 `application.yml` 中配置 Redis 和 Caffeine：
```yaml
spring:
  redis:
  host: 127.0.0.1
  port: 6379
  lettuce:
    pool:
      max-active: 8
      max-idle: 8
      min-idle: 0
      max-wait: 100ms
  caffeine:
    spec: maximumSize=1000,expireAfterWrite=5m
```

#### （3）实现多级缓存管理
多级缓存需要通过手动管理，无法仅依赖注解。我们可以通过创建缓存工具类实现多级缓存逻辑。

```java
@Component
public class MultiLevelCache {

    private final Cache<String, Object> caffeineCache;
    private final RedisTemplate<String, Object> redisTemplate;

    public MultiLevelCache(RedisTemplate<String, Object> redisTemplate) {
        this.caffeineCache = Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(5, TimeUnit.MINUTES)
                .build();
        this.redisTemplate = redisTemplate;
    }

    public Object get(String key) {
        // 1. 查询本地缓存
        Object value = caffeineCache.getIfPresent(key);
        if (value != null) {
            return value;
        }

        // 2. 查询 Redis 缓存
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            // 同步到本地缓存
            caffeineCache.put(key, value);
        }
        return value;
    }

    public void put(String key, Object value) {
        // 更新本地缓存
        caffeineCache.put(key, value);
        // 更新 Redis 缓存
        redisTemplate.opsForValue().set(key, value);
    }

    public void evict(String key) {
        // 清除本地缓存
        caffeineCache.invalidate(key);
        // 清除 Redis 缓存
        redisTemplate.delete(key);
    }
}
```

#### （4）整合多级缓存到业务逻辑
修改 `UserService`，整合多级缓存：

```java
@Service
public class UserService {

    @Autowired
    private final MultiLevelCache multiLevelCache;

    private final Map<Long, User> database = new HashMap<>();

    public User getUserById(Long id) {
        String key = "user:" + id;
        User user = (User) multiLevelCache.get(key);
        if (user == null) {
            System.out.println("Fetching user from database...");
            user = database.get(id);
            if (user != null) {
                multiLevelCache.put(key, user);
            }
        }
        return user;
    }

    public User updateUser(User user) {
        String key = "user:" + user.getId();
        System.out.println("Updating user in database...");
        database.put(user.getId(), user);
        multiLevelCache.put(key, user);
        return user;
    }

    public void deleteUser(Long id) {
        String key = "user:" + id;
        System.out.println("Deleting user from database...");
        database.remove(id);
        multiLevelCache.evict(key);
    }
}
```

[Redis]:https://redis.io/
[Caffeine]:https://www.baeldung.com/java-caching-caffeine
