---
title: 使用Spring AOP + 自定义注解管理Neo4j Java Driver事务
published: 2024-10-31
description: ''
image: "./zhu.png"
tags: [Java, Spring Boot, Neo4j]
category: 'Neo4j'
draft: false
lang: 'zh_CN'
---

# 使用Spring AOP与自定义注解管理Neo4j Java Driver事务

## 事务管理概述
事务是数据库操作中的一个关键概念，它保证了一组操作要么全部成功，要么全部失败，从而保证数据的完整性和一致性。事务具备以下四大特性（ACID）：
1. **原子性（Atomicity）**: 事务是一个不可分割的工作单位，事务中的操作要么全部完成，要么全部不完成，系统不会处于不一致的状态。
2. **一致性（Consistency）**: 事务必须使数据库从一个一致性状态变到另一个一致性状态，确保数据的完整性。
3. **原子性（Atomicity）**: 事务的执行是独立的，一个事务的执行不会受到其他事务的干扰。
4. **持久性（Durability）**: 一旦事务提交，其对数据库的更改将是永久的，即使系统发生故障也不会丢失。

数据库开发中，事务管理对于数据的安全性和一致性至关重要。尤其在分布式系统中，事务的管理更加复杂，需要通过合理的设计来保障系统的一致性。

## Spring 框架的@Transcational注解
[@Transactional][@Transactional] 注解是 Spring 框架提供的一个用于管理事务的注解。它可以应用于方法或类级别，确保方法中的数据库操作在一个事务中执行。`@Transactional` 可以与多种数据访问技术（[MyBatis](https://mybatis.org/mybatis-3/)、[JPA](https://www.oracle.com/java/technologies/persistence-jsp.html)、[Spring Data Neo4j](https://docs.spring.io/spring-data/neo4j/reference/getting-started.html)）结合使用，帮助我们简化事务的管理。
`@Transactional` 可以与这些框架无缝集成。例如，当你在使用 `MyBatis` 的 `DAO` 或 `Mapper` 方法时，可以在服务层的方法上使用 `@Transactional` 注解，确保多个数据库操作在一个事务中执行。

## Neo4j Java Driver的事务
[Neo4j Java Transactions](https://neo4j.com/docs/java-manual/current/transactions/) 中，只有在 [Session][Session] 中才能保证事务的正确性和有效性，事务中的查询将作为一个整体执行，或者根本不执行。在使用 Neo4j Java Driver 时，事务是与 Session 绑定的，所以我们不能在Spring Boot中使用 `@Transactional` 注解来管理 Neo4j Java Driver 事务，必须使用 Session 来手动管理事务。

## 读操作中的事务
使用 `neo4j-java-driver` 进行读操作时，事务的范围可以根据实际需求进行灵活控制。可以选择将事务范围限定在具体的查询操作上，而不是整个业务方法。这样可以提高代码的可读性和性能，避免不必要的事务开销。

### 添加neo4j-java-driver依赖
```xml
<dependency>
    <groupId>org.neo4j.driver</groupId>
    <artifactId>neo4j-java-driver</artifactId>
    <version>4.4.9</version>
</dependency>
```

### 业务层代码
```java
public Optional<NodeVO> getNodeByUuid(final String uuid) {
    final NodeVO graphNode = nodeMapper.getNodeByUuid(uuid);
    return Optional.ofNullable(graphNode);
}
```

### 数据访问层代码（读操作）
```java
public NodeVO getNodeByUuid(final String uuid) {
    final String cypherQuery = "MATCH (n:GraphNode { uuid: $uuid }) RETURN n";

    try (Session session = driver.session(SessionConfig.builder().build())) {
        return session.readTransaction(tx -> {
            final var result = tx.run(cypherQuery, Values.parameters(Constants.UUID, uuid));

            NodeVO n = null;
            while (result.hasNext()) {
                final Record record = result.next();
                n = nodeExtractor.extractNode(record.get("n"));
            }

            return n;
        });
    }
}
```

### Node提取方法
```java
public NodeVO extractNode(final Object node) {
    final NodeVO nodeInfo = new NodeVO();

    if (node instanceof NodeValue) {
        final NodeValue nodeValue = (NodeValue) node;
        final Map<String, Object> nodeMap = nodeValue.asNode().asMap();
        final Map<String, String> stringNodeMap = nodeMap.entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey, entry -> String.valueOf(entry.getValue())));

        setNodeInfo(stringNodeMap, nodeInfo);
    }
    return nodeInfo;
}
```

在 Neo4j 中，使用 `readTransaction` 方法可以在只读操作中提供一种简单的事务管理机制，以确保在该操作期间不会发生数据变化，从而提供了一种一致性保障。

## 写操作中的事务
Neo4j Java Driver 提供了 `writeTransaction` 方法，用于在写操作中开启事务。在 Neo4j 中，使用此方法可以在进行写操作时提供一种简单的事务管理机制，以确保在该操作期间不会发生数据变化，从而提供了一种一致性保障。

### 业务层代码
```java
@Autowired
private Driver neo4jDriver;

public void updateNode(final NodeUpdateDTO nodeUpdateDTO) {
    try (Session session = neo4jDriver.session()) {
        session.writeTransaction(tx -> {
            final String uuid = nodeUpdateDTO.getUuid();
            final Optional<NodeVO> graphNodeByUuid = getNodeByUuid(uuid);
            final String current = getCurrentTime();

            if (graphNodeByUuid.isPresent()) {
                nodeMapper.updateNodeByUuid(nodeUpdateDTO, current, tx);
            } else {
                final String message = String.format(Message.NODE_NULL, uuid);
                LOG.error(message);
                throw new NoSuchElementException(message);
            }
            return null;
        });
    } catch (Exception e) {
        LOG.error("Error updating node: {}", e.getMessage(), e);
        throw new RuntimeException("Error updating node", e);
    }
}
```

### 数据访问层代码（写操作）
```java
public void updateNodeByUuid(final NodeUpdateDTO nodeUpdateDTO, final String currentTime, final Transaction tx) {
    final StringBuilder setProperties = getSetProperties(nodeUpdateDTO.getProperties().entrySet());

    final String cypherQuery = "MATCH (gn:GraphNode {uuid: $nodeUuid}) "
            + "SET gn = { uuid: gn.uuid, "
            + "create_time: gn.create_time, "
            + "update_time: $updateTime"
            + setProperties
            + " }";

    tx.run(cypherQuery, Values.parameters(
            Constants.NODE_UUID, nodeUpdateDTO.getUuid(),
            Constants.UPDATE_TIME, currentTime));
}
```

通过在 `updateNode` 上开启一个 `Session` 级别的事务，将 `Transaction tx` 传递给 `nodeMapper`，我们可以确保整个业务流程在事务内对数据库的操作是原子性的，要么全部执行，要么全部不执行。
我们也可以通过  结合自定义注解的方式来管理事务，从而避免在每个服务方法中显式地使用 Session 和事务。这可以使代码更加整洁，减少重复代码。

## 使用Spring AOP和自定义注解简化事务管理
为了避免每个服务方法都显式地管理事务，我们可以结合[Spring AOP](https://docs.spring.io/spring-framework/reference/core/aop.html)和自定义注解来自动化事务管理，从而避免在每个服务方法中显式地使用 Session 和事务，使代码更加简洁，事务的控制也变得更加模块化。

### 自定义注解
首先，我们定义一个自定义注解`@Neo4jTransactional`，用于标识需要进行事务管理的方法。
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Neo4jTransactional {
}
```

### Spring AOP
AOP是Spring框架的核心机制之一，旨在通过将 `横切关注点（cross-cutting concerns）` 从业务逻辑中分离出来，提高代码的模块化程度。横切关注点是指那些影响多个模块的功能，例如日志记录、事务管理、安全检查等。AOP通过将这些横切关注点封装在独立的模块中，使得业务逻辑更加清晰和简洁。
我们可以将 Neo4j Java 的 `Session` 抽取出来，以注解的方式作用到目标的 `Service` 方法上，实现事务管理。

#### 添加Spring AOP依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

#### 切面类
```java
@Aspect
@Component
@Order(1)
public class Neo4jTransactionAspect {

    private static final Logger LOG = LoggerFactory.getLogger(Neo4jTransactionAspect.class);

    @Resource
    private TransactionManager neo4jTransactionManager;

    @Autowired
    private Session neo4jSession;

    /**
     * Handles Neo4j transactions.
     * @param joinPoint Join point
     * @return The result of the method call
     * @throws Throwable if an error occurs
     */
    @Around("@annotation(com.paiondata.aristotle.common.annotion.Neo4jTransactional)")
    @SuppressWarnings({"checkstyle:IllegalThrows", "checkstyle:IllegalCatch"})
    public Object manageTransaction(final ProceedingJoinPoint joinPoint) throws Throwable {
        Transaction tx = null;
        try {
            tx = neo4jSession.beginTransaction();

            final Object[] args = joinPoint.getArgs();
            final MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            final Method method = signature.getMethod();
            final Parameter[] parameters = method.getParameters();

            // Inject the transaction
            injectTransaction(tx, args, parameters);

            final Object result = joinPoint.proceed(args);

            neo4jTransactionManager.commitTransaction(tx);
            return result;
        } catch (final Exception e) {
            if (tx != null) {
                neo4jTransactionManager.rollbackTransaction(tx);
            }

            LOG.error(String.format("Transaction error: %s", e.getMessage()), e);
            throw new IllegalStateException("Something went wrong inside Aristotle webservice. "
                    + "Please file an issue at https://github.com/paion-data/aristotle/issues to report this incident. "
                    + "We apologize for the inconvenience", e);
        } finally {
            closeTransaction(tx);
        }
    }

    /**
     * Inject the transaction into the method arguments.
     * @param tx Transaction
     * @param args the method arguments
     * @param parameters the method parameters
     */
    private void injectTransaction(final Transaction tx, final Object[] args, final Parameter[] parameters) {
        boolean transactionInjected = false;
        for (int i = 0; i < parameters.length; i++) {
            if (parameters[i].getType().equals(Transaction.class)) {
                args[i] = tx;
                transactionInjected = true;
                break;
            }
        }

        if (!transactionInjected) {
            final String message = Message.METHOD_WITHOUT_TRANSACTION;
            LOG.error(message);
            throw new IllegalArgumentException(message);
        }
    }

    /**
     * Close the transaction.
     * @param tx Transaction
     */
    private void closeTransaction(final Transaction tx) {
        if (tx != null && tx.isOpen()) {
            tx.close();
        }
    }
}
```

### 将注解添加到业务方法上

```java
/**
 * Updates a graph node based on the provided DTO.
 * <p>
 * Extracts the UUID from the provided {@code nodeUpdateDTO}.
 * Retrieves the graph node by the extracted UUID using the {@link #getNodeByUuid(String)} method.
 * If the node is found, it updates the node using <br>
 * the {@link NodeMapper#updateNodeByUuid(NodeUpdateDTO, String, Transaction)} method.
 * If the node is not found, it throws a {@link NoSuchElementException} with an error message including the UUID.
 *
 * @param nodeUpdateDTO the DTO containing information for updating the node. <br>
 * It includes the UUID and other update parameters.
 * @param tx            the transaction object used for the database operation
 * @throws NoSuchElementException if the node with the specified UUID is not found in the graph
 */
@Neo4jTransactional
@Override
public void updateNode(final NodeUpdateDTO nodeUpdateDTO, final Transaction tx) {
    final String uuid = nodeUpdateDTO.getUuid();
    final Optional<NodeVO> graphNodeByUuid = getNodeByUuid(uuid);
    final String current = getCurrentTime();

    if (graphNodeByUuid.isPresent()) {
        nodeMapper.updateNodeByUuid(nodeUpdateDTO, current, tx);
    } else {
        final String message = String.format(Message.NODE_NULL, uuid);
        LOG.error(message);
        throw new NoSuchElementException(message);
    }
}
```

### 在数据访问层中使用 Transaction tx
```java
/**
     * Updates a graph node by its UUID.
     * <p>
     * Constructs a Cypher query to match a graph node by its UUID and update its properties.
     * The query dynamically includes only the fields that need to be updated based on the provided properties.
     * Executes the Cypher query using the provided transaction.
     *
     * @param nodeUpdateDTO the DTO containing the updated properties of the node
     * @param currentTime the current timestamp for the update time
     * @param tx the Neo4j transaction to execute the Cypher query
     */
    @Override
    public void updateNodeByUuid(final NodeUpdateDTO nodeUpdateDTO, final String currentTime, final Transaction tx) {
        final StringBuilder setProperties = getSetProperties(nodeUpdateDTO.getProperties().entrySet());

        final String cypherQuery = "MATCH (gn:GraphNode {uuid: $nodeUuid}) "
                + "SET gn = { uuid: gn.uuid, "
                + "create_time: gn.create_time, "
                + "update_time: $updateTime"
                + setProperties
                + " }";

        tx.run(cypherQuery, Values.parameters(
                Constants.NODE_UUID, nodeUpdateDTO.getUuid(),
                Constants.UPDATE_TIME, currentTime));
    }
```

实现的关键点在于我们需要确保 `Transaction` 参数能够在方法中传递，最后开启Cypher查询。

#### 为什么一定要使用切面类中的Transaction来查询

因为在 `Neo4j Java` 中，一个 [Session][Session] 可以链接多个事务，但在任何时候，一个 `Session` 中只能有一个活动的事务。为了维护多个并发事务，需要使用多个并发 `Session`。所以我们需要确保切面类中的 `Transaction` 参数能够传递，否则在切面类中开启的事务将无法被其他方法使用。

### 在控制器中传递参数

在切面类中，我们将 `Transaction` 参数传递给业务方法，导致业务方法必须在方法签名中包含 `Transaction` 参数。导致我们必须要在控制器中传递一个 `null` 参数来占位，这降低了代码的可读性和可维护性。

```java
/**
 * Updates a node.
 *
 * <p>
 * This method handles a POST request to update a node based on the provided update DTO.
 * It validates the input DTO and calls the node service to perform the update.
 * The result is wrapped in a {@link Result} object with a success message.
 *
 * @param nodeUpdateDTO the {@link NodeUpdateDTO} containing the updated node information
 * @return a {@link Result} object containing a success message
 */
@ApiOperation(value = "Updates a node")
@PostMapping("/update")
public Result<String> updateNode(@RequestBody @Valid final NodeUpdateDTO nodeUpdateDTO) {
    nodeService.updateNode(nodeUpdateDTO, null);
    return Result.ok(Message.UPDATE_SUCCESS);
}
```

**这个问题不能被很好的解决，如果你有更好的解决方案，欢迎在[issue](https://github.com/paion-data/aristotle/issues/36)中评论。**

[@Transactional]: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html
[Session]: https://neo4j.com/docs/api/java-driver/current/org.neo4j.driver/org/neo4j/driver/Driver.html#session()
