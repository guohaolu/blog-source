# 消费者组内的分区分配详解

1. **同组消费者分区分配**
```plaintext
场景一：单主题多分区
Topic-A (3个分区)
┌─────────────┐
│ Partition 0 │────► Consumer1
│ Partition 1 │────► Consumer2
│ Partition 2 │────► Consumer1
└─────────────┘

Consumer Group X
├── Consumer1: 消费Partition 0,2
└── Consumer2: 消费Partition 1

特点：
- 每个消费者确实消费不同分区
- 但这些分区的数据合起来是主题的全量数据
- 消费者组作为整体可以看到主题的所有数据
```

2. **不同订阅的消费者能否在同组**
```plaintext
场景二：订阅主题不同
Consumer1订阅：Topic A,B,C
Consumer2订阅：Topic A,B,C,D

结论：不能在同一组！
原因：
1. 违反了消费者组的订阅原则
2. 可能导致数据消费混乱
3. Kafka会抛出异常

正确做法：
Consumer Group X
├── Consumer1: 订阅 A,B,C,D
└── Consumer2: 订阅 A,B,C,D

或者分成不同组：
Group1: Consumer1 (A,B,C)
Group2: Consumer2 (A,B,C,D)
```

3. **实际案例说明**
```java
// 错误示例：同组不同订阅
public class WrongGroupExample {
    public static void main(String[] args) {
        // Consumer1
        KafkaConsumer<String, String> consumer1 = new KafkaConsumer<>(props);
        consumer1.subscribe(Arrays.asList("A", "B", "C"));
        
        // Consumer2 (同组但订阅不同)
        KafkaConsumer<String, String> consumer2 = new KafkaConsumer<>(props);
        consumer2.subscribe(Arrays.asList("A", "B", "C", "D"));
        
        // 会导致异常！
    }
}

// 正确示例：同组相同订阅
public class CorrectGroupExample {
    public static void main(String[] args) {
        // 所有消费者订阅相同的主题
        List<String> topics = Arrays.asList("A", "B", "C", "D");
        
        // Consumer1
        KafkaConsumer<String, String> consumer1 = new KafkaConsumer<>(props);
        consumer1.subscribe(topics);
        
        // Consumer2
        KafkaConsumer<String, String> consumer2 = new KafkaConsumer<>(props);
        consumer2.subscribe(topics);
    }
}
```

4. **消费者组的数据完整性**
```plaintext
示例场景：
Topic-A (4个分区)
┌─────────────┐
│ Partition 0 │──┐
│ Partition 1 │──┼─► Consumer Group X
│ Partition 2 │──┤  ├── Consumer1
│ Partition 3 │──┘  └── Consumer2
└─────────────┘

数据流：
1. 生产者发送消息到各个分区
2. 分区0,1 分配给Consumer1
3. 分区2,3 分配给Consumer2
4. 消费者组作为整体消费了所有数据
```

5. **关键原则**
```plaintext
1. 订阅原则
   - 同组消费者必须订阅相同主题
   - 违反会导致异常

2. 分区分配
   - 分区是最小分配单位
   - 一个分区只能分配给一个消费者
   - 消费者组能看到所有数据

3. 扩展性
   - 增加消费者可以提高并行度
   - 消费者数不应超过分区总数
``` 