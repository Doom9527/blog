---
title: 在Spring Boot中使用AOP + 自定义注解管理Neo4j Java Driver事务
published: 2024-10-31
description: ''
image: "./zhu.png"
tags: [Java, Spring Boot, Neo4j]
category: 'Neo4j'
draft: false
lang: 'zh_CN'
---

# 在Spring Boot中使用AOP + 自定义注解管理Neo4j Java Driver事务

## Spring 框架的@Transcational注解
[@Transactional]((https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)) 注解是 Spring 框架提供的一个用于管理事务的注解。它可以应用于方法或类级别，确保方法中的数据库操作在一个事务中执行。`@Transactional` 可以与多种数据访问技术一起使用，包括 [MyBatis](https://mybatis.org/mybatis-3/)、[JPA](https://www.oracle.com/java/technologies/persistence-jsp.html)、[Spring Data Neo4j](https://docs.spring.io/spring-data/neo4j/reference/getting-started.html) 等。
`@Transactional` 可以与这些框架无缝集成。例如，当你在使用 `MyBatis` 的 `DAO` 或 `Mapper` 方法时，可以在服务层的方法上使用 `@Transactional` 注解，确保多个数据库操作在一个事务中执行。

## Neo4j Java Driver的事务