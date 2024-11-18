---
title: 分布式图数据缓存加载器
published: 2024-11-12
description: ''
image: './pic-01.jpg'
tags: [Java, Redis, Neo4j]
category: 'Cache'
draft: false 
lang: 'zh_CN'
---

# 分布式 Neo4j 数据库缓存加载器

在分布式知识图谱的应用场景中，[Neo4j](https://neo4j.com/docs/) 数据库中的大规模图数据计算是常见需求，但随着节点数量的增加，计算性能和响应速度会受到很大挑战。为了解决这些问题，我们需要设计一个 **分布式缓存加载器**，使用 **BFS** 算法计算图的最大深度扩展，并将计算结果存储到 Redis 中，作为缓存服务，提高后续查询性能。

## 一、需求分析

在业务场景中，每个节点需要支持**最大深度扩展图**的计算，主要涉及以下需求：

### 1.最大深度扩展计算：
通过 BFS 遍历节点关系，计算出图的扩展信息（节点与关系），并存储最大深度信息。

### 2.性能瓶颈：
随着 Neo4j 数据库中节点数达到百万量级，单次查询计算变得非常缓慢，无法在用户请求时即时完成。

### 3.Redis 缓存：
使用 Redis 存储扩展图及其深度信息，将计算好的数据缓存起来，减少后续的重复计算。

### 4.定时任务与断点续传：

- 每天凌晨重新加载所有节点数据到 Redis。
- 在新任务开始时，若上一个任务未完成，则强制终止上一个任务。
- 支持断点续传，通过记录 Redis 中的进度避免重复加载。

## 二、整体设计方案

### （1）任务调度
- 使用 [Spring Schedule][Spring Schedule] 实现定时任务，每天 0 点触发加载。
- 在任务启动时检查 Redis 中的当前任务进度，如果存在未完成任务，从断点继续执行。

### （2）节点遍历与图计算
- 使用 Neo4j 的 Cypher 查询，批量加载节点：
```cypher
MATCH (n:Node) RETURN n ORDER BY id(n) SKIP $start LIMIT $batchSize
```
- 在 Java 应用中使用 **BFS** 算法 遍历图，计算扩展图及最大深度。

### （3）缓存存储
- 将每个节点的 UUID、扩展图（节点 + 关系数据）以及最大深度存储到 Redis：
```java
redisTemplate.opsForValue().set("graph:{nodeUuid}:data", graphData);
redisTemplate.opsForValue().set("graph:{nodeUuid}:maxDepth", maxDepth);
```

### （4）进度管理
- Redis 中维护当前任务的进度：
```java
redisTemplate.opsForValue().set("task:progress", currentProgress, 1, TimeUnit.HOURS);
```

- 每次更新节点数据后，记录当前进度，任务完成时清理进度记录。

## 三、实现细节
### （1）定时任务初始化
使用 Spring 的 [@Scheduled](https://www.baeldung.com/spring-scheduled-tasks) 注解实现每天 0 点触发的定时任务：
```java
@Slf4j
@Component
public class CacheLoaderScheduler {

    @Autowired
    private CacheLoaderService cacheLoaderService;

    @Scheduled(cron = "0 0 0 * * ?")
    public void reloadCache() {
        log.info("Starting cache reload task...");
        cacheLoaderService.startLoading();
    }
}
```

### （2）缓存加载服务
设计一个服务类，用于执行分布式缓存加载逻辑：
```java
@Slf4j
@Service
public class CacheLoaderService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private Neo4jService neo4jService;

    private volatile boolean isRunning = false;

    public synchronized void startLoading() {
        if (isRunning) {
            log.warn("Previous task is still running, forcing termination...");
            isRunning = false;
        }
        isRunning = true;

        try {
            // 从 Redis 获取上次任务的进度
            Integer start = (Integer) redisTemplate.opsForValue().get("task:progress");
            if (start == null) {
                start = 0;
            }

            log.info("Starting task from progress: {}", start);

            // 批量加载节点并计算图
            final int batchSize = 25;
            while (isRunning) {
                List<NodeVO> nodes = neo4jService.loadNodes(start, batchSize);
                if (nodes.isEmpty()) {
                    log.info("All nodes processed. Task completed.");
                    redisTemplate.delete("task:progress");
                    break;
                }

                for (NodeVO node : nodes) {
                    if (!isRunning) break;

                    // BFS 遍历图并存储到 Redis
                    processNode(node);

                    // 更新任务进度
                    start++;
                    redisTemplate.opsForValue().set("task:progress", start, 1, TimeUnit.HOURS);
                }
            }
        } finally {
            isRunning = false;
        }
    }

    private void processNode(NodeVO node) {
        // 使用 BFS 计算扩展图和最大深度
        Graph graph = neo4jService.kDegreeExpansion(node.getGraphUuid(), node.getUuid(), -1);

        // 存储到 Redis
        redisTemplate.opsForValue().set("graph:" + node.getUuid() + ":data", graph);
        redisTemplate.opsForValue().set("graph:" + node.getUuid() + ":maxDepth", graph.getMaxDepth());

        log.info("Processed node: {}", node.getUuid());
    }
}
```

### （3）Neo4j 数据操作
在 `Neo4jService` 中实现节点加载和 BFS 图计算逻辑：
```java
@Service
public class Neo4jService {

    @Autowired
    private Driver driver;

    public List<NodeVO> loadNodes(int start, int batchSize) {
        final String query = "MATCH (n:Node) RETURN n ORDER BY id(n) SKIP $start LIMIT $batchSize";
        try (Session session = driver.session(SessionConfig.builder().build())) {
            var result = session.run(query, Values.parameters("start", start, "batchSize", batchSize));
            List<NodeVO> nodes = new ArrayList<>();
            while (result.hasNext()) {
                Record record = result.next();
                nodes.add(NodeVO.from(record.get("n").asNode()));
            }
            return nodes;
        }
    }

    public Graph kDegreeExpansion(String graphUuid, String nodeUuid, int k) {
        // 省略具体实现
    }
}
```

## 四、Redis 数据结构
### Redis 存储内容：
#### 1.扩展图：
每个节点的扩展图存储在 `graph:{nodeUuid}:data`：
```json
{
  "nodes": [...],
  "relations": [...],
  "maxDepth": 5
}
```

#### 2.任务进度：
当前任务进度存储在 `task:progress`。

### 设置超时时间：
- 每次更新 `task:progress` 时，设置 1 小时过期时间，避免任务意外终止后长期占用。

## 五、总结
通过以上设计，实现了一个基于 `Neo4j` 和 `Redis` 的分布式缓存加载器，能够高效地计算节点扩展图并将结果缓存至 Redis。关键点在于：

**1.断点续传**： 在任务中断后能够从 Redis 中记录的进度继续执行，避免重复计算。
**2.任务抢占**： 支持强制终止上一个未完成任务，保证每天任务准时执行。
**3.批量加载与存储**： 使用分页加载和批量存储机制提升性能，减少对 Neo4j 和 Redis 的压力。

[Spring Schedule]: https://spring.io/guides/gs/scheduling-tasks
