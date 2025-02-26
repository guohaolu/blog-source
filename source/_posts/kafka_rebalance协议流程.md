# Kafka Rebalance协议详解

1. **核心角色**
```plaintext
参与者：
┌─────────────────┐    ┌─────────────────┐
│  Group          │    │     Consumer    │
│  Coordinator    │    │     Group       │
│  (协调者)        │    │     (消费组)    │
└─────────────────┘    └─────────────────┘

- Coordinator：负责管理消费组的组件，运行在Broker上
- Consumer Group：消费者组的所有成员
```

2. **Join Group阶段**
```plaintext
步骤1：所有消费者向Coordinator发送JoinGroup请求
┌─────────┐         ┌──────────┐
│Consumer1│─────┐   │          │
└─────────┘     │   │          │
┌─────────┐     ├──>│Coordinator
│Consumer2│─────┤   │          │
└─────────┘     │   │          │
┌─────────┐     │   │          │
│Consumer3│─────┘   │          │
└─────────┘         └──────────┘

步骤2：Coordinator选择一个消费者作为Leader
┌─────────┐         ┌──────────┐
│Consumer1│◄────────│          │
└─────────┘(Leader) │          │
┌─────────┐         │Coordinator
│Consumer2│◄────────│          │
└─────────┘         │          │
┌─────────┐         │          │
│Consumer3│◄────────│          │
└─────────┘         └──────────┘
```

3. **Sync Group阶段**
```plaintext
步骤1：Leader消费者制定分区分配方案
┌─────────┐
│Consumer1│ 制定分配方案：
└─────────┘ Consumer1 -> Partition[0,1]
  (Leader)  Consumer2 -> Partition[2,3]
           Consumer3 -> Partition[4,5]

步骤2：Leader将方案发送给Coordinator
┌─────────┐         ┌──────────┐
│Consumer1│────────>│          │
└─────────┘         │Coordinator
                    │          │
                    └──────────┘

步骤3：Coordinator将方案下发给所有消费者
┌─────────┐         ┌──────────┐
│Consumer1│◄────────│          │
└─────────┘         │          │
┌─────────┐         │Coordinator
│Consumer2│◄────────│          │
└─────────┘         │          │
┌─────────┐         │          │
│Consumer3│◄────────│          │
└─────────┘         └──────────┘
```

4. **Heartbeat阶段**
```plaintext
所有消费者定期发送心跳
┌─────────┐   心跳   ┌──────────┐
│Consumer1│─────────>│          │
└─────────┘         │          │
┌─────────┐         │Coordinator
│Consumer2│─────────>│          │
└─────────┘         │          │
┌─────────┐         │          │
│Consumer3│─────────>│          │
└─────────┘         └──────────┘

心跳超时：
- session.timeout.ms：心跳超时时间
- heartbeat.interval.ms：心跳发送间隔
- max.poll.interval.ms：消息处理超时时间
```

5. **完整流程示例**
```java
// 消费者配置
Properties props = new Properties();
props.put("group.id", "my-group");
props.put("session.timeout.ms", "10000");
props.put("heartbeat.interval.ms", "3000");

// Rebalance监听器
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    // 再均衡开始前
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // 提交偏移量
        consumer.commitSync(currentOffsets);
    }
    
    // 再均衡完成后
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // 初始化新分区的消费位置
        for (TopicPartition partition : partitions) {
            consumer.seek(partition, getLastOffset(partition));
        }
    }
}); 
```