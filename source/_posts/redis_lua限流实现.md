# Redis+Lua分布式限流实现

1. **基本架构**
```plaintext
请求流转图：
┌──────────┐    1.请求     ┌──────────┐
│ Client 1 │─────────────►│          │
└──────────┘              │          │
┌──────────┐    2.限流    │   API    │
│ Client 2 │◄────────────►│ Gateway  │
└──────────┘              │          │
┌──────────┐    3.计数    │          │
│ Client 3 │─────────────►│          │
└──────────┘              └────┬─────┘
                              │
                         4.执行Lua脚本
                              │
                              ▼
                        ┌──────────┐
                        │  Redis   │
                        └──────────┘
```

2. **Lua限流脚本**
```lua
-- 限流脚本
-- KEYS[1]: 限流key
-- ARGV[1]: 时间窗口大小(秒)
-- ARGV[2]: 限流阈值
-- ARGV[3]: 当前时间戳
local key = KEYS[1]
local window = tonumber(ARGV[1])
local threshold = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- 1. 移除时间窗口之前的数据
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

-- 2. 获取当前窗口的请求数
local count = redis.call('ZCARD', key)

-- 3. 判断是否超过阈值
if count >= threshold then
    return 0
end

-- 4. 记录本次请求
redis.call('ZADD', key, now, now)

-- 5. 设置过期时间
redis.call('EXPIRE', key, window)

return 1
```

3. **Java实现**
```java
public class RedisLimiter {
    private final StringRedisTemplate redisTemplate;
    private final String luaScript;
    
    public RedisLimiter(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
        // 加载Lua脚本
        this.luaScript = loadLuaScript();
    }
    
    public boolean isAllowed(String key, int window, int threshold) {
        List<String> keys = Collections.singletonList(key);
        long now = System.currentTimeMillis();
        
        // 执行Lua脚本
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            keys,
            String.valueOf(window),
            String.valueOf(threshold),
            String.valueOf(now)
        );
        
        return result != null && result == 1;
    }
}
```

4. **动态调整实现**
```java
public class DynamicRateLimiter {
    private static final String LIMIT_CONFIG_KEY = "rate:limit:config";
    
    // 更新限流阈值
    public void updateThreshold(String key, int threshold) {
        redisTemplate.opsForHash().put(
            LIMIT_CONFIG_KEY, 
            key, 
            String.valueOf(threshold)
        );
    }
    
    // 获取当前阈值
    private int getCurrentThreshold(String key) {
        String value = (String) redisTemplate.opsForHash()
            .get(LIMIT_CONFIG_KEY, key);
        return value == null ? 
            defaultThreshold : Integer.parseInt(value);
    }
    
    // 限流检查
    public boolean isAllowed(String key) {
        int threshold = getCurrentThreshold(key);
        return redisLimiter.isAllowed(
            key, 
            window, 
            threshold
        );
    }
}
```

5. **滑动时间窗口示意**
```plaintext
时间窗口滑动：
     now-60s          now
        │             │
        ▼             ▼
┌─────────────────────┐
│   时间窗口(60s)     │
└─────────────────────┘
        │             │
        │  ┌──────┐   │
        │  │请求量│   │
        │  └──────┘   │
        │             │
  过期的请求    新请求
```

6. **使用示例**
```java
@RestController
public class ApiController {
    private final DynamicRateLimiter limiter;
    
    @GetMapping("/api/test")
    public String test() {
        String key = "api:test";
        if (!limiter.isAllowed(key)) {
            throw new RuntimeException("请求被限流");
        }
        // 业务逻辑
        return "success";
    }
    
    @PostMapping("/limit/update")
    public void updateLimit(
        @RequestParam String key, 
        @RequestParam int threshold
    ) {
        limiter.updateThreshold(key, threshold);
    }
}
``` 