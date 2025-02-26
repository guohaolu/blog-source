# Kafka消费者组(Consumer Group)详解

1. **基本概念**
```plaintext
消费者组定义：
- 多个消费者实例组成的逻辑分组
- 共同消费一个或多个主题
- 每个消费者组有唯一的group.id

消费模型：
┌─────────────┐
│   Topic-A   │
│ Partition 0 │────┐
│ Partition 1 │──┐ │
│ Partition 2 │┐ │ │    Consumer Group X
└─────────────┘│ │ │  ┌─────────────────┐
               │ │ └──►│   Consumer 1    │
               │ └────►│   Consumer 2    │
               └──────►│   Consumer 3    │
                      └─────────────────┘
```

2. **关键特性**
```plaintext
1. 分区所有权
   - 一个分区只能被组内一个消费者消费
   - 一个消费者可以消费多个分区

2. 负载均衡
   - 组内成员自动分配分区
   - 支持动态扩缩容

3. 消费位置管理
   - 每个组独立维护消费位置
   - 支持从任意位置开始消费

4. 故障转移
   - 自动检测消费者故障
   - 自动重新分配分区
```

3. **消费者组状态**
```plaintext
状态流转：
Empty ──► PreparingRebalance ──► CompletingRebalance ──► Stable
  ▲                                                        │
  └────────────────────────────────────────────────────────┘

主要状态：
1. Empty: 组内无成员
2. PreparingRebalance: 准备开始重平衡
3. CompletingRebalance: 完成分区分配
4. Stable: 稳定状态，正常消费
```

4. **配置示例**
```java
// 消费者组配置
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-group");           // 组ID
props.put("enable.auto.commit", "true");     // 自动提交
props.put("auto.commit.interval.ms", "1000"); // 提交间隔
props.put("session.timeout.ms", "30000");    // 会话超时
props.put("max.poll.interval.ms", "300000"); // 最大轮询间隔

// 创建消费者
KafkaConsumer<String, String> consumer = 
    new KafkaConsumer<>(props);

// 订阅主题
consumer.subscribe(Arrays.asList("my-topic"));
```

5. **消费示例**
```java
public class GroupConsumerExample {
    public static void main(String[] args) {
        KafkaConsumer<String, String> consumer = 
            new KafkaConsumer<>(props);
            
        try {
            while (true) {
                // 批量拉取消息
                ConsumerRecords<String, String> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                // 处理消息
                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);
                }
                
                // 异步提交位移
                consumer.commitAsync();
            }
        } finally {
            try {
                // 同步提交最后的位移
                consumer.commitSync();
            } finally {
                consumer.close();
            }
        }
    }
}
```

6. **常见使用场景**
```plaintext
1. 消息队列模式
   - 一个分区一个消费者
   - 组内负载均衡
   
2. 发布订阅模式
   - 多个消费者组
   - 每个组都收到全量消息

示例：
Topic-A ──► Consumer Group 1 (日志处理)
       └──► Consumer Group 2 (数据分析)
       └──► Consumer Group 3 (监控告警)
```

7. **监控指标**
```plaintext
关键指标：
1. 消费延迟(lag)
2. 消费速率
3. 提交失败率
4. 重平衡频率
5. 活跃消费者数
``` 