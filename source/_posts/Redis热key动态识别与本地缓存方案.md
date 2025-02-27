---
title: Redis热key动态识别与本地缓存方案
date: 2024-03-21 14:00:00
categories:
  - 中间件
tags:
  - Redis
  - 缓存
  - 性能优化
---

## 热key识别方案

### 1. 客户端采样统计

```java
@Slf4j
public class HotKeyDetector {
    private LoadingCache<String, LongAdder> keyCounterCache = CacheBuilder.newBuilder()
        .expireAfterWrite(1, TimeUnit.MINUTES)
        .build(new CacheLoader<String, LongAdder>() {
            @Override
            public LongAdder load(String key) {
                return new LongAdder();
            }
        });
    
    // 采样率1%
    private static final double SAMPLE_RATE = 0.01;
    
    public void recordKeyAccess(String key) {
        // 采样统计
        if (ThreadLocalRandom.current().nextDouble() < SAMPLE_RATE) {
            keyCounterCache.getUnchecked(key).increment();
        }
    }
    
    // 定时任务，每分钟统计热key
    @Scheduled(fixedRate = 60000)
    public void detectHotKeys() {
        Map<String, LongAdder> counters = keyCounterCache.asMap();
        // 按访问量排序，取Top N
        List<Map.Entry<String, LongAdder>> hotKeys = counters.entrySet().stream()
            .sorted((e1, e2) -> Long.compare(e2.getValue().sum(), e1.getValue().sum()))
            .limit(100)
            .collect(Collectors.toList());
            
        // 推送到本地缓存
        updateLocalCache(hotKeys);
    }
}
```

### 2. Redis Server端监控

```ascii
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│Redis Monitor│--->│日志分析服务  │--->│热key计算    │
└─────────────┘    └─────────────┘    └─────────────┘
                                            │
                                            ▼
                                    ┌─────────────┐
                                    │本地缓存更新  │
                                    └─────────────┘
```

```bash
# Redis MONITOR命令采样
redis-cli MONITOR | grep -v "PING" | awk '{print $4}' | sort | uniq -c | sort -nr | head -n 10
```

### 3. 代理层统计

```java
@Aspect
@Component
public class RedisAccessAspect {
    private static final int WINDOW_SIZE_SECONDS = 60;
    private TimeWindowCounter counter = new TimeWindowCounter(WINDOW_SIZE_SECONDS);
    
    @Around("execution(* org.springframework.data.redis.core.RedisTemplate.*(..))")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        String key = extractKey(point);
        counter.increment(key);
        return point.proceed();
    }
}

public class TimeWindowCounter {
    private Queue<Map<String, AtomicInteger>> windows = new LinkedList<>();
    private final int windowSize;
    
    public void increment(String key) {
        getCurrentWindow().computeIfAbsent(key, k -> new AtomicInteger()).incrementAndGet();
    }
    
    public Map<String, Integer> getTopKeys(int n) {
        // 合并所有时间窗口的统计数据
        return mergeWindows().entrySet().stream()
            .sorted((e1, e2) -> e2.getValue().compareTo(e1.getValue()))
            .limit(n)
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                (v1, v2) -> v1,
                LinkedHashMap::new
            ));
    }
}
```

## 大key识别方案

### 1. Redis命令扫描

```bash
# 使用SCAN命令渐进式扫描
redis-cli --bigkeys -i 0.1
```

### 2. 自定义扫描工具

```java
@Service
public class BigKeyScanner {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public List<KeySize> scanBigKeys(int threshold) {
        List<KeySize> bigKeys = new ArrayList<>();
        ScanOptions options = ScanOptions.scanOptions().count(100).build();
        Cursor<String> cursor = redisTemplate.scan(options);
        
        while(cursor.hasNext()) {
            String key = cursor.next();
            long size = getKeySize(key);
            if (size > threshold) {
                bigKeys.add(new KeySize(key, size));
            }
        }
        return bigKeys;
    }
    
    private long getKeySize(String key) {
        DataType type = redisTemplate.type(key);
        switch (type) {
            case STRING:
                return redisTemplate.opsForValue().get(key).toString().length();
            case HASH:
                return redisTemplate.opsForHash().size(key);
            case LIST:
                return redisTemplate.opsForList().size(key);
            case SET:
                return redisTemplate.opsForSet().size(key);
            case ZSET:
                return redisTemplate.opsForZSet().size(key);
            default:
                return 0;
        }
    }
}
```

## 本地缓存优化方案

### 1. 多级缓存架构

```ascii
请求 --> 本地缓存(Caffeine) --> Redis集群 --> 数据库
     │                      │
     │                      │
     └──────────────────────┘
          热key直接返回
```

### 2. 本地缓存实现

```java
@Service
public class MultiLevelCache {
    private LoadingCache<String, Object> localCache = Caffeine.newBuilder()
        .maximumSize(10000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .recordStats()
        .build(key -> null); // 缓存未命中时返回null
        
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public Object get(String key) {
        // 1. 查询本地缓存
        Object value = localCache.getIfPresent(key);
        if (value != null) {
            return value;
        }
        
        // 2. 查询Redis
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            // 如果是热key，放入本地缓存
            if (isHotKey(key)) {
                localCache.put(key, value);
            }
        }
        
        return value;
    }
    
    private boolean isHotKey(String key) {
        // 从热key统计结果中判断
        return HotKeyDetector.isHot(key);
    }
}
```

### 3. 缓存一致性保证

```java
@Service
public class CacheConsistencyManager {
    @Autowired
    private MultiLevelCache multiLevelCache;
    
    // 监听数据变更消息
    @KafkaListener(topics = "cache-update")
    public void handleCacheUpdate(CacheUpdateMessage message) {
        // 删除本地缓存
        multiLevelCache.evict(message.getKey());
        // 更新Redis缓存
        multiLevelCache.refreshRedis(message.getKey());
    }
}
```

## 监控指标

```yaml
metrics:
  - name: "hot_key_count"
    type: "gauge"
    labels:
      - "key"
      - "qps"
      
  - name: "big_key_size"
    type: "gauge"
    labels:
      - "key"
      - "size"
      
  - name: "local_cache_hit_rate"
    type: "gauge"
    labels:
      - "application"
```

## 最佳实践

1. **采样率控制**
   - 客户端采样率动态调整
   - 重点监控高QPS时段

2. **本地缓存策略**
   - 仅缓存热key
   - 设置合理的过期时间
   - 控制缓存数量

3. **一致性保证**
   - 消息队列通知更新
   - 定时刷新机制
   - 版本号控制

4. **监控告警**
   - 热key变化趋势
   - 内存使用监控
   - 缓存命中率监控 