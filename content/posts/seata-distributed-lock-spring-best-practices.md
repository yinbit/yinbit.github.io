+++
date = '2026-04-17T20:00:00+08:00'
draft = false
title = 'Spring生态中使用Seata分布式锁的最佳实践教程'
tags = ["Seata", "分布式锁", "Spring Boot", "微服务", "分布式事务"]
categories = ["技术"]
+++

## Spring生态中使用Seata分布式锁的最佳实践教程

在分布式系统中，分布式锁是保证数据一致性和并发控制的重要工具。Seata作为一款优秀的分布式事务解决方案，除了提供强大的分布式事务能力外，还内置了全局锁机制。本文将详细介绍如何在Spring生态中使用Seata的分布式锁功能。

## 一、Seata分布式锁概述

### 1.1 什么是Seata全局锁

Seata的全局锁（Global Lock）是AT模式下实现事务隔离的核心机制。它主要用于防止分布式事务中的脏写问题，确保在全局事务未完成前，其他事务无法修改同一数据[^1]。

Seata全局锁的特点：
- **互斥性**：同一时间只有一个全局事务能持有特定数据的全局锁
- **高可用性**：依托Seata TC（事务协调器）集群实现高可用
- **自动管理**：全局锁的获取和释放由Seata框架自动处理

### 1.2 Seata全局锁的实现原理

Seata的全局锁实现建立在其AT模式的数据源代理（DataSourceProxy）之上。当执行SQL操作时，Seata会：

1. 在一阶段执行SQL时，解析SQL生成锁键（lock key）
2. 向TC注册分支事务时获取全局锁
3. 只有获取全局锁成功后，才允许提交本地事务
4. 在二阶段提交或回滚时，释放全局锁[^1]

## 二、Seata全局锁与@GlobalLock注解

### 2.1 @GlobalLock注解的作用

Seata提供了`@GlobalLock`注解，这是一个轻量级的工具，适用于不需要开启全局事务，但需要检查全局锁以避免脏读脏写的场景[^1]。

与`@GlobalTransactional`相比，`@GlobalLock`的优势：
- **轻量级**：不需要开启全局事务，减少RPC调用开销
- **简单高效**：只检查全局锁是否存在，不参与事务协调
- **适用场景广**：可用于读取操作、单服务更新等场景

### 2.2 @GlobalLock的使用场景

`@GlobalLock`主要用于以下场景：

1. **查询操作**：需要读取可能被全局事务修改的数据，防止脏读
2. **独立更新**：某些数据更新操作不需要参与分布式事务，但需要避免被全局事务回滚影响
3. **混合场景**：部分业务逻辑使用全局事务，部分逻辑使用本地事务但需要锁保护

## 三、Spring Boot中集成Seata

### 3.1 环境准备

首先，我们需要在Spring Boot项目中添加Seata依赖：

```xml
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    <version>2022.0.0.0-RC2</version>
    <exclusions>
        <exclusion>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 3.2 配置Seata

在`application.yml`中配置Seata：

```yaml
seata:
  enabled: true
  application-id: order-service
  tx-service-group: my_test_tx_group
  registry:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace:
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace:
  service:
    vgroup-mapping:
      my_test_tx_group: default
    grouplist:
      default: 127.0.0.1:8091
```

### 3.3 数据库准备

在每个参与事务的数据库中创建`undo_log`表：

```sql
CREATE TABLE `undo_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `branch_id` bigint NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
```

## 四、Seata分布式锁的实战应用

### 4.1 使用@GlobalTransactional注解

在需要分布式事务的方法上使用`@GlobalTransactional`注解，这会自动开启全局锁：

```java
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    
    @Resource
    private OrderMapper orderMapper;
    @Resource
    private StorageFeignClient storageFeignClient;
    @Resource
    private PaymentFeignClient paymentFeignClient;

    /**
     * 创建订单，包含扣减库存和创建支付记录
     */
    @Override
    @GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
    public Order createOrder(Order order) {
        log.info("开始创建订单: {}", order);
        
        // 1. 创建订单
        order.setStatus(0);
        orderMapper.insert(order);
        log.info("订单创建成功: {}", order.getId());
        
        try {
            // 2. 扣减库存
            storageFeignClient.decrease(order.getProductId(), order.getCount());
            log.info("库存扣减成功: 商品ID={}, 数量={}", order.getProductId(), order.getCount());
            
            // 3. 创建支付记录
            Payment payment = new Payment();
            payment.setUserId(order.getUserId());
            payment.setOrderId(order.getId());
            payment.setAmount(order.getMoney());
            payment.setStatus(0);
            paymentFeignClient.create(payment);
            log.info("支付记录创建成功: {}", payment.getId());
            
            // 4. 更新订单状态为已完成
            order.setStatus(1);
            orderMapper.updateById(order);
            log.info("订单状态更新为已完成: {}", order.getId());
            
            return order;
        } catch (Exception e) {
            log.error("创建订单失败，将回滚事务", e);
            throw new RuntimeException("创建订单失败", e);
        }
    }
}
```

### 4.2 使用@GlobalLock注解

在不需要分布式事务但需要全局锁保护的场景下使用`@GlobalLock`：

```java
@Service
@Slf4j
public class ProductServiceImpl implements ProductService {
    
    @Resource
    private ProductMapper productMapper;

    /**
     * 查询商品信息，使用@GlobalLock防止脏读
     */
    @Override
    @GlobalLock
    public Product getProductById(Long productId) {
        log.info("查询商品信息: productId={}", productId);
        Product product = productMapper.selectById(productId);
        log.info("商品信息查询完成: {}", product);
        return product;
    }

    /**
     * 更新商品库存，使用@GlobalLock防止脏写
     */
    @Override
    @GlobalLock
    public void updateStock(Long productId, Integer count) {
        log.info("更新商品库存: productId={}, count={}", productId, count);
        Product product = productMapper.selectById(productId);
        if (product == null) {
            throw new RuntimeException("商品不存在");
        }
        product.setStock(product.getStock() + count);
        productMapper.updateById(product);
        log.info("商品库存更新完成: productId={}, stock={}", productId, product.getStock());
    }
}
```

### 4.3 配合select for update使用

对于需要更强隔离性的场景，可以结合`select for update`和`@GlobalLock`使用：

```java
@Service
@Slf4j
public class InventoryServiceImpl implements InventoryService {
    
    @Resource
    private InventoryMapper inventoryMapper;

    /**
     * 扣减库存，使用select for update和@GlobalLock
     */
    @Override
    @GlobalLock
    public void decreaseStock(Long productId, Integer count) {
        log.info("开始扣减库存: productId={}, count={}", productId, count);
        
        // 使用select for update获取行锁
        Inventory inventory = inventoryMapper.selectForUpdate(productId);
        if (inventory == null) {
            throw new RuntimeException("库存记录不存在");
        }
        if (inventory.getStock() < count) {
            throw new RuntimeException("库存不足");
        }
        
        // 扣减库存
        inventory.setStock(inventory.getStock() - count);
        inventoryMapper.updateById(inventory);
        
        log.info("库存扣减成功: productId={}, remainingStock={}", productId, inventory.getStock());
    }
}
```

对应的Mapper：

```java
public interface InventoryMapper extends BaseMapper<Inventory> {
    
    /**
     * 使用select for update查询库存
     */
    @Select("SELECT * FROM t_inventory WHERE product_id = #{productId} FOR UPDATE")
    Inventory selectForUpdate(@Param("productId") Long productId);
}
```

## 五、Seata全局锁的工作机制

### 5.1 防止脏写的机制

让我们通过一个场景来理解Seata全局锁如何防止脏写：

**脏写场景**：
1. 业务一开启全局事务，修改数据A
2. 业务二修改数据A
3. 业务一发生异常需要回滚，但数据已被业务二修改

**使用全局锁后的流程**：
1. 业务一执行SQL，在提交前向TC获取全局锁
2. 业务二尝试修改数据A，在获取全局锁时发现已被占用，进行重试
3. 如果业务一需要回滚，而业务二持有数据库锁，不会造成死锁
4. 业务二重试获取全局锁失败后，进行事务回滚
5. 业务一获得锁并成功回滚[^2]

### 5.2 处理非Seata管理的事务

对于没有交给Seata管理的事务，Seata提供了另一种保护机制：

1. 当Seata事务更新数据时，会记录更新前后的快照
2. 回滚时，会比较当前数据和Seata修改后的快照
3. 如果相同则正常回滚
4. 如果不同则记录异常、发送警告，并进行人工介入[^2]

## 六、Seata与Redisson的结合使用

在某些场景下，我们可以将Seata和Redisson结合使用，发挥各自的优势。

### 6.1 高并发事务场景

在高并发的事务场景（如促销活动）中，可以使用Redisson的分布式锁控制并发量，防止大量请求同时进入Seata事务：

```java
@Service
@Slf4j
public class PromotionServiceImpl implements PromotionService {
    
    @Resource
    private RedissonClient redissonClient;
    @Resource
    private OrderService orderService;
    
    private static final String PROMOTION_LOCK_KEY = "promotion:lock:";

    /**
     * 促销活动下单，结合分布式锁和分布式事务
     */
    @Override
    public Order createPromotionOrder(Order order) {
        // 获取促销活动锁，限制并发量
        RLock lock = redissonClient.getLock(PROMOTION_LOCK_KEY + order.getProductId());
        
        try {
            // 尝试获取锁，控制并发数量
            boolean locked = lock.tryLock(500, 5, TimeUnit.MILLISECONDS);
            if (!locked) {
                throw new RuntimeException("活动太火爆，请稍后再试");
            }
            
            // 调用订单服务，内部使用Seata分布式事务
            return orderService.createOrder(order);
        } catch (InterruptedException e) {
            log.error("创建促销订单时发生中断", e);
            Thread.currentThread().interrupt();
            throw new RuntimeException("创建订单失败，请重试");
        } finally {
            // 释放锁
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

## 七、最佳实践总结

### 7.1 使用Seata全局锁的最佳实践

1. **合理选择注解**：
   - 需要分布式事务时使用`@GlobalTransactional`
   - 只需要锁保护时使用`@GlobalLock`
   - 两者都会检查全局锁，但`@GlobalLock`更轻量

2. **事务设计原则**：
   - 保持事务短小精悍，避免长时间持有锁
   - 合理设置事务超时时间
   - 避免在事务中执行远程调用或耗时操作

3. **锁粒度控制**：
   - 尽量使用行级锁，避免表级锁
   - 合理设计锁键，提高并发度

4. **异常处理**：
   - 正确处理锁冲突异常
   - 设置合理的重试策略
   - 监控锁等待和超时情况

### 7.2 性能优化建议

1. **选择合适的存储模式**：
   - 测试环境可用File模式
   - 生产环境建议使用DB或Redis模式
   - 对性能要求高且已有Redis的场景可用Redis模式

2. **TC集群部署**：
   - 生产环境部署TC集群，避免单点故障
   - 使用Nacos等注册中心实现服务发现

3. **监控和告警**：
   - 监控全局锁的等待时间和获取成功率
   - 设置合理的告警阈值

## 八、总结

Seata的全局锁机制是其AT模式实现事务隔离的核心，通过`@GlobalTransactional`和`@GlobalLock`注解，我们可以在Spring生态中方便地使用这一功能。

在实际项目中，我们应该根据业务场景选择合适的方案：
- 需要保证多个操作原子性的场景使用`@GlobalTransactional`
- 只需要防止脏读脏写的场景使用`@GlobalLock`
- 高并发场景可以考虑结合Redisson等工具进行流量控制

通过合理使用Seata的全局锁功能，我们可以在保证数据一致性的同时，获得较好的性能表现。

[^1]: [详解Seata AT 模式事务隔离级别与全局锁设计](https://seata.apache.org/zh-cn/blog/seata-at-lock/)
[^2]: [Seate分布式锁](https://blog.csdn.net/qq_54421200/article/details/139687195)
