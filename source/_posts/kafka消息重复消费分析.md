# Kafka消息重复消费分析

1. **重复消费场景分类**
```plaintext
场景分类：
1. 生产端重复发送
2. 消费端重复消费
3. Rebalance导致重复
4. 位移提交异常

影响程度：
┌────────────────┬───────────┬────────────┐
│    场景类型    │ 影响范围  │  发生概率   │
├────────────────┼───────────┼────────────┤
│生产端重复      │ 单条消息  │   低       │
│消费端重复      │ 批量消息  │   中       │
│Rebalance重复   │ 分区数据  │   高       │
│位移提交异常    │ 批量消息  │   中       │
└────────────────┴───────────┴────────────┘
```

2. **生产端重复发送**
```java
// 问题代码
public void send(String topic, String message) {
    try {
        producer.send(new ProducerRecord<>(topic, message));
    } catch (Exception e) {
        // 捕获异常后重试，可能导致重复发送
        producer.send(new ProducerRecord<>(topic, message));
    }
}

// 解决方案：启用幂等性
Properties props = new Properties();
props.put("enable.idempotence", true);
props.put("transactional.id", "tx-1");
// 使用事务API
producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(record);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

3. **消费端重复消费**
```java
// 问题代码
@KafkaListener(topics = "my-topic")
public void consume(ConsumerRecord<String, String> record) {
    processMessage(record);  // 处理消息
    consumer.commitSync();   // 同步提交可能失败
}

// 解决方案：实现幂等消费
public class IdempotentConsumer {
    private final Set<String> processedMessages = 
        Collections.synchronizedSet(new HashSet<>());
    
    @KafkaListener(topics = "my-topic")
    public void consume(ConsumerRecord<String, String> record) {
        String messageId = generateMessageId(record);
        if (processedMessages.add(messageId)) {
            try {
                processMessage(record);
                // 持久化消息ID
                saveMessageId(messageId);
            } catch (Exception e) {
                processedMessages.remove(messageId);
                throw e;
            }
        }
    }
}
```

4. **Rebalance导致重复**
```java
// 问题：Rebalance时未提交的消息会重复消费
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // 分区被撤销时可能未及时提交位移
    }
});

// 解决方案：优化Rebalance配置和提交策略
Properties props = new Properties();
// 增加会话超时时间，减少不必要的Rebalance
props.put("session.timeout.ms", "30000");
// 使用手动提交，确保处理完成后再提交
props.put("enable.auto.commit", "false");

// 实现再均衡监听器
class SaveOffsetsOnRebalance implements ConsumerRebalanceListener {
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Rebalance前确保提交位移
        consumer.commitSync(currentOffsets);
    }
}
```

5. **位移提交异常**
```java
// 问题代码
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
    try {
        consumer.commitSync(); // 批量提交可能失败
    } catch (Exception e) {
        // 异常处理不当导致重复消费
    }
}

// 解决方案：实现精确的位移管理
public class OffsetManager {
    private final Map<TopicPartition, OffsetAndMetadata> offsets = 
        new ConcurrentHashMap<>();
    
    public void markOffset(String topic, int partition, long offset) {
        offsets.put(
            new TopicPartition(topic, partition),
            new OffsetAndMetadata(offset + 1)
        );
    }
    
    public void commitOffsets(KafkaConsumer<?, ?> consumer) {
        try {
            consumer.commitSync(offsets);
            offsets.clear();
        } catch (Exception e) {
            // 记录失败的位移，下次重试
            handleCommitFailure(offsets, e);
        }
    }
}
```

6. **最佳实践**
```plaintext
1. 消息设计
   - 生成全局唯一消息ID
   - 包含业务去重字段
   - 添加时间戳信息

2. 消费端优化
   - 实现幂等性检查
   - 使用本地缓存+持久化存储
   - 合理设置批处理大小

3. 监控告警
   - 监控重复消费率
   - 设置消费延迟阈值
   - 跟踪位移提交状态
``` 