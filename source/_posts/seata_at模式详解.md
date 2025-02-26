# Seata AT模式详解

1. **AT模式原理**
```plaintext
执行流程：
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   业务SQL    │     │   解析SQL   │     │  生成回滚SQL │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 记录前镜像   │ ──► │  执行业务SQL │ ──► │ 记录后镜像   │
└─────────────┘     └─────────────┘     └─────────────┘

两阶段提交：
Phase 1: Try
- 业务SQL执行
- 解析SQL
- 记录前后镜像
- 生成回滚日志

Phase 2: Commit/Rollback
- Commit: 删除回滚日志
- Rollback: 根据镜像生成反向SQL
```

2. **核心组件**
```java
// 1. 数据源代理
@Bean
public DataSource dataSource() {
    DruidDataSource dataSource = new DruidDataSource();
    // 配置数据源
    dataSource.setUrl("jdbc:mysql://localhost:3306/test");
    
    // 使用Seata代理数据源
    return new DataSourceProxy(dataSource);
}

// 2. 全局事务注解
@GlobalTransactional
@Transactional
public void businessMethod() {
    // 订单服务
    orderService.create(order);
    // 库存服务
    stockService.deduct(stock);
    // 账户服务
    accountService.debit(money);
}
```

3. **实际应用场景**
```plaintext
场景一：电商下单
┌─────────────┐
│  下单服务   │
└──────┬──────┘
       │
       ▼
┌──────┴──────┐     ┌─────────────┐     ┌─────────────┐
│  订单服务   │ ──► │  库存服务   │ ──► │  账户服务   │
└─────────────┘     └─────────────┘     └─────────────┘
订单表         库存表         账户表
- 创建订单     - 扣减库存     - 扣减余额

场景二：积分兑换
┌─────────────┐
│  兑换服务   │
└──────┬──────┘
       │
       ▼
┌──────┴──────┐     ┌─────────────┐
│  积分服务   │ ──► │  商品服务   │
└─────────────┘     └─────────────┘
积分表         库存表
- 扣减积分     - 扣减库存
```

4. **实现示例**
```java
// 订单服务
@GlobalTransactional
public void createOrder(OrderDTO orderDTO) {
    // 1. 创建订单
    Order order = new Order();
    order.setUserId(orderDTO.getUserId());
    order.setAmount(orderDTO.getAmount());
    orderMapper.insert(order);
    
    // 2. 调用库存服务
    Result stockResult = stockService.deduct(
        orderDTO.getProductId(), 
        orderDTO.getQuantity()
    );
    if (!stockResult.isSuccess()) {
        throw new BusinessException("库存不足");
    }
    
    // 3. 调用账户服务
    Result accountResult = accountService.debit(
        orderDTO.getUserId(), 
        orderDTO.getAmount()
    );
    if (!accountResult.isSuccess()) {
        throw new BusinessException("余额不足");
    }
}
```

5. **优缺点分析**
```plaintext
优点：
1. 对业务无侵入
2. 无需手动编写回滚逻辑
3. 性能损耗小
4. 兼容已有数据库

缺点：
1. 依赖数据库事务
2. 需要代理数据源
3. 无法处理复杂业务
4. 表需要主键和索引
```

6. **使用建议**
```plaintext
适用场景：
1. 简单的数据库操作
2. 常规的CRUD业务
3. 对性能要求不是特别高
4. 数据库支持事务

不适用场景：
1. 复杂业务逻辑
2. 高并发场景
3. 涉及非事务性资源
4. 跨数据库类型
```

7. **性能优化**
```java
@Configuration
public class SeataConfig {
    
    @Bean
    public GlobalTransactionScanner globalTransactionScanner() {
        return new GlobalTransactionScanner(
            "order-service",  // 应用名
            "my_test_tx_group"  // 事务分组
        );
    }
    
    // 配置事务分组
    @Bean
    public ConfigurationCache configurationCache() {
        ConfigurationCache cache = new ConfigurationCache();
        cache.putConfig(
            "service.vgroupMapping.my_test_tx_group",
            "default"
        );
        return cache;
    }
}
``` 