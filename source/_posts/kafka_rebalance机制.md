# Kafka Rebalance机制详解

1. **触发Rebalance的场景**
```plaintext
1. 消费组成员变更
   - 新消费者加入消费组
   - 消费者主动离开消费组
   - 消费者崩溃离开消费组

2. Topic分区数变更
   - 增加分区
   - 管理员手动分区重分配

3. 订阅Topic数变更
   - 消费者订阅新Topic
   - 正则表达式订阅匹配新Topic
```

2. **Rebalance协议流程**
```plaintext
第一阶段：Join Group
┌─────────┐     ┌─────────┐
│Consumer1│     │Group    │
│Consumer2│ --> │Coordinator
│Consumer3│     │         │
└─────────┘     └─────────┘
发送JoinGroup请求

第二阶段：Sync Group
┌─────────┐     ┌─────────┐
│Leader   │     │Group    │
│Consumer │ --> │Coordinator
│         │     │         │
└─────────┘     └─────────┘
制定分配方案

第三阶段：Heart Beat
定期发送心跳保持分配方案
```

3. **分区分配策略**
```java
// 1. RangeAssignor (默认)
public class RangeAssignor {
    // 按照分区序号范围划分
    // 例如：6个分区，2个消费者
    // consumer1: [0,1,2]
    // consumer2: [3,4,5]
}

// 2. RoundRobinAssignor
public class RoundRobinAssignor {
    // 轮询分配
    // consumer1: [0,2,4]
    // consumer2: [1,3,5]
}

// 3. StickyAssignor
public class StickyAssignor {
    // 粘性分配，尽量保持原有分配
    // 减少分区迁移
}
```

4. **性能影响**
```plaintext
Rebalance过程中：
1. 消费者停止消费
2. 释放分区所有权
3. 重新分配分区
4. 重新建立连接
5. 从新位置开始消费

影响：
- 消费延迟增加
- 消费者暂时不可用
- 可能重复消费
```

5. **优化建议**
```java
// 1. 合理配置超时时间
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 10000);
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);

// 2. 选择合适的分配策略
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG, 
    StickyAssignor.class.getName());

// 3. 优雅关闭消费者
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    consumer.wakeup();
    consumer.close();
}));
```

6. **监控指标**
```plaintext
关键指标：
1. Rebalance频率
2. Rebalance持续时间
3. 消费延迟变化
4. 分区分配不均衡度

告警阈值：
- Rebalance频率 > 10次/小时
- Rebalance时间 > 10秒
- 消费延迟突增 > 1000条
``` 