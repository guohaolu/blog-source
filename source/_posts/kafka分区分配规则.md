# Kafka分区分配规则详解

1. **分配的基本原则**
```plaintext
关键规则：
1. 消费者只能消费其订阅的主题
2. 一个分区只能分配给同一个消费组中的一个消费者
3. 分配是以<主题,分区>为单位进行的

示例：
Consumer1订阅：topic2
Consumer2订阅：topic1
┌─────────┐     ┌─────────┐
│Topic1   │     │Consumer1│
│Partition1│ ╳   │(topic2)│  ╳ 不会分配
└─────────┘     └─────────┘
                
┌─────────┐     ┌─────────┐
│Topic1   │     │Consumer2│
│Partition1│ →   │(topic1)│  ✓ 会分配
└─────────┘     └─────────┘
```

2. **订阅模式**
```java
// 1. 直接订阅指定主题
consumer.subscribe(Arrays.asList("topic1", "topic2"));

// 2. 正则表达式订阅
consumer.subscribe(Pattern.compile("topic.*"));

// 3. 手动分配分区
consumer.assign(Arrays.asList(
    new TopicPartition("topic1", 0),
    new TopicPartition("topic1", 1)
));
```

3. **分配过程示例**
```plaintext
场景：
- Topic1: 3个分区
- Topic2: 2个分区
- Consumer1: 订阅Topic2
- Consumer2: 订阅Topic1
- Consumer3: 订阅Topic1, Topic2

最终分配：
Consumer1 <- Topic2-P0, Topic2-P1
Consumer2 <- Topic1-P0, Topic1-P1, Topic1-P2
Consumer3 <- Topic2-P0, Topic2-P1, Topic1-P0, Topic1-P1, Topic1-P2

注意：Consumer3因为订阅了两个主题，
所以可能同时获得两个主题的分区
```

4. **验证代码**
```java
public class KafkaSubscriptionTest {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "test-group");
        
        // Consumer1只订阅Topic2
        KafkaConsumer<String, String> consumer1 = 
            new KafkaConsumer<>(props);
        consumer1.subscribe(Arrays.asList("topic2"));
        
        // 获取分配的分区
        Set<TopicPartition> assignment = consumer1.assignment();
        for (TopicPartition partition : assignment) {
            // 只会看到Topic2的分区
            System.out.println(partition.topic() + 
                             "-" + partition.partition());
        }
    }
}
``` 