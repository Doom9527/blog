---
title: 内存加载器
published: 2024-11-12
description: ''
image: ''
tags: [Java, Redis, Neo4j]
category: 'Redis'
draft: false 
lang: 'zh_CN'
---

## Neo4j数据库连接

**[Neo4j Java Driver][Neo4j Java Driver]**：使用 Neo4j Java Driver 来与 Neo4j 数据库进行连接，并执行图查询。

## 数据存储到Redis

**Redis**：将所有节点数据存储到 Redis 中，作为缓存层。这部分操作不涉及查询，只是将节点和扩展后的节点和数据存入 Redis。
   - Redis Hashes 或 Sets：存储节点UUID和展开最大深度以及可能的扩展数据。
   - Redis 过期策略：不设置过期时间，在每天加载新数据时清除 Redis 中的相关数据。

### 使用 RedisTemplate 操作 Redis

考虑到目前场景对性能要求、并发操作不高，选择 Spring 高度封装的 [RedisTemplate](https://docs.spring.io/spring-data/redis/reference/redis/template.html)。

## 定时任务与调度

**[Spring Scheduler](https://spring.io/guides/gs/scheduling-tasks)**：使用 Spring 的 `@Scheduled` 注解来设置定时任务，每天 0 点时执行加载操作，重新从 Neo4j 加载数据并刷新 Redis 中的节点信息。

## 数据加载
   - 搜索最大深度：使用 BFS 算法确定每个节点的展开最大深度，将最大深度和节点 UUID 存入 Redis。
   - 加载扩展数据：在加载数据时，使用 BFS 算法从指定的起始节点开始，遍历并扩展到最大深度的节点。当扩展的节点数超过设定的阈值时，将这些数据存入 Redis。

[Neo4j Java Driver]: https://neo4j.com/docs/java-manual/current/install/
