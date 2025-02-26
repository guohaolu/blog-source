# Rebalance后消费位置重置的问题

1. **可能带来的问题**
```plaintext
场景示例：
Consumer1原本消费Partition0：
┌───────────────────────────┐
│     Partition 0          │
│ offset: 1000000 -> 1500000│
└───────────────────────────┘

Rebalance后分配给Consumer2：
1. 需要初始化消费位置
2. 可能需要建立新的TCP连接
3. 重新填充消费者缓存
4. 可能导致重复消费
```

2. **性能影响**
```plaintext
主要开销：
1. 状态重建
   - 重新加载消费位置
   - 重建内部缓存
   
2. 网络开销
   - 建立新的TCP连接
   - 首次拉取数据
   
3. 消费延迟
   - 处理暂停
   - 消息积压增加
```

3. **优化方案**
```java
// 1. 使用StickyAssignor策略
properties.put(
    ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    "org.apache.kafka.clients.consumer.StickyAssignor"
);

// 2. 合理配置缓存大小
properties.put(
    ConsumerConfig.FETCH_MAX_BYTES_CONFIG,
    "5242880"  // 5MB
);

// 3. 实现优雅的偏移量管理
class MyConsumer {
    private Map<TopicPartition, OffsetAndMetadata> currentOffsets 
        = new ConcurrentHashMap<>();
        
    @KafkaListener(topics = "my-topic")
    public void consume(ConsumerRecord<String, String> record) {
        // 处理消息
        processMessage(record);
        
        // 记录偏移量
        currentOffsets.put(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1)
        );
    }
    
    // Rebalance监听器
    class SaveOffsetOnRebalance implements ConsumerRebalanceListener {
        public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
            // Rebalance前提交偏移量
            consumer.commitSync(currentOffsets);
        }
    }
}
```

4. **最佳实践**
```plaintext
1. 减少Rebalance频率
   - 合理设置session.timeout.ms
   - 避免频繁重启消费者

2. 优化消费者配置
   - 使用StickyAssignor
   - 合理设置fetch.max.bytes
   - 启用消费者缓存

3. 监控指标
   - 消费延迟
   - Rebalance频率
   - 处理时间
```

5. **新版本优化(Kafka 2.4+)**
```plaintext
增量式Rebalance：
1. 只对变更的分区进行重分配
2. 未变更的分区继续消费
3. 显著减少重平衡影响

Cooperative Rebalancing：
1. 两阶段提交过程
2. 允许消费者继续处理未撤销的分区
3. 减少消费中断时间
``` 