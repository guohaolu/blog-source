---
title: Redis缓存击穿问题的紧急处理与最佳实践
date: 2024-03-21 11:00:00
categories:
  - 中间件
tags:
  - Redis
  - 缓存
  - 性能优化
  - 故障处理
---

## 问题背景

在一次线上故障中，某热门商品的缓存key恰好过期，导致大量请求直接击穿到数据库，引发系统性能问题。本文将详细分析处理过程和解决方案。

## 故障现象

```ascii
┌──────────┐         ┌──────────┐         ┌──────────┐
│  用户请求 │ ──────> │  Redis   │ ──────> │  数据库  │
└──────────┘         └──────────┘         └──────────┘
     │                    ×                    !!!
     │                缓存失效              QPS暴增
     │                                    响应变慢
     └─────────────────────────────────────────┘
              大量请求直接访问数据库
```

主要表现：
1. Redis某个key突然失效
2. 大量并发请求涌入数据库
3. 数据库CPU使用率飙升
4. 系统响应时间显著增加

## 紧急处理流程

### 1. 数据库限流保护

```java
@Slf4j
public class DbProtector {
    private RateLimiter rateLimiter = RateLimiter.create(100.0); // 限制QPS为100

    public Product queryProduct(Long productId) {
        if (!rateLimiter.tryAcquire()) {
            log.warn("数据库访问被限流，productId: {}", productId);
            throw new RuntimeException("系统繁忙，请稍后重试");
        }
        return productMapper.selectById(productId);
    }
}
```

### 2. 问题商品下线

```ascii
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ 运维平台     │--->│ 配置中心    │--->│ 应用服务    │
└─────────────┘    └─────────────┘    └─────────────┘
      │                                      │
      │                                      │
      └──────────────────────────────────────┘
             更新商品状态为"已下线"
```

### 3. 手动Mock缓存

```java
@Service
public class CacheRecoveryService {
    @Autowired
    private RedisTemplate redisTemplate;
    
    public void mockProductCache(Long productId, Product product) {
        String cacheKey = "product:" + productId;
        // 设置较短的过期时间，便于后续恢复
        redisTemplate.opsForValue().set(cacheKey, product, 5, TimeUnit.MINUTES);
        log.info("Mock cache success for productId: {}", productId);
    }
}
```

### 4. 重启服务

```ascii
分批重启流程：
┌────────────┐     ┌────────────┐     ┌────────────┐
│  实例1下线  │ --> │  实例2下线  │ --> │  实例3下线  │
└────────────┘     └────────────┘     └────────────┘
      │                  │                  │
      ▼                  ▼                  ▼
┌────────────┐     ┌────────────┐     ┌────────────┐
│  实例1上线  │ --> │  实例2上线  │ --> │  实例3上线  │
└────────────┘     └────────────┘     └────────────┘
```

## 长期解决方案

### 1. 缓存预热

```java
@Component
public class CacheWarmer {
    @Scheduled(cron = "0 0 3 * * ?")  // 每天凌晨3点执行
    public void warmHotProducts() {
        List<Long> hotProductIds = getHotProductIds();
        for (Long productId : hotProductIds) {
            Product product = productService.getById(productId);
            cacheService.setProductCache(productId, product);
        }
    }
}
```

### 2. 双重检查锁防击穿

```java
public Product getProduct(Long productId) {
    String cacheKey = "product:" + productId;
    Product product = redisTemplate.opsForValue().get(cacheKey);
    
    if (product == null) {
        String lockKey = "lock:" + productId;
        try {
            // 获取分布式锁
            if (redisTemplate.opsForValue().setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS)) {
                // 双重检查
                product = redisTemplate.opsForValue().get(cacheKey);
                if (product == null) {
                    product = productMapper.selectById(productId);
                    redisTemplate.opsForValue().set(cacheKey, product, 1, TimeUnit.HOURS);
                }
            }
        } finally {
            redisTemplate.delete(lockKey);
        }
    }
    return product;
}
```

### 3. 缓存降级方案

```ascii
正常访问流程：
┌──────────┐     ┌──────────┐     ┌──────────┐
│   请求    │ --> │  Redis   │ --> │  数据库   │
└──────────┘     └──────────┘     └──────────┘

降级后流程：
┌──────────┐     ┌──────────┐     ┌──────────┐
│   请求    │ --> │ 本地缓存  │ --> │  数据库   │
└──────────┘     └──────────┘     └──────────┘
```

## 监控预警

1. **缓存监控指标**：
   - 缓存命中率
   - 缓存过期监控
   - 数据库QPS监控

2. **告警规则**：
```yaml
rules:
  - name: "缓存击穿告警"
    conditions:
      - metric: "cache.miss.rate"
        threshold: 80%  # 缓存未命中率超过80%
      - metric: "db.qps"
        threshold: 1000 # 数据库QPS超过1000
    duration: "1m"     # 持续1分钟
    severity: "critical"
```

## 经验总结

1. **预防措施**：
   - 热点数据永不过期
   - 定时缓存预热
   - 多级缓存设计

2. **应急处理**：
   - 及时限流保护
   - 快速恢复服务
   - 分批重启降低影响

3. **长期规划**：
   - 完善监控体系
   - 建立降级方案
   - 优化缓存策略 