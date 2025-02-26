# Kafka的ISR和OSR机制

1. **基本概念**
```plaintext
AR (Assigned Replicas): 所有副本
ISR (In-Sync Replicas): 同步副本集合
OSR (Out-of-Sync Replicas): 非同步副本集合

关系：AR = ISR + OSR
```

2. **ISR(In-Sync Replicas)**
```plaintext
特点：
- 与leader保持同步的follower集合
- 包含leader副本自身
- 动态调整：follower可能进入或退出ISR

判断标准：
1. replica.lag.time.max.ms内有同步请求
2. 副本落后leader的消息数不超过replica.lag.max.messages
```

3. **OSR(Out-of-Sync Replicas)**
```plaintext
特点：
- 与leader副本同步滞后的follower集合
- 暂时无法参与副本选举
- 会尝试追赶leader数据

产生原因：
1. 网络延迟或阻塞
2. follower所在broker负载过高
3. follower崩溃或重启
```

4. **动态维护机制**
```java
// 伪代码展示ISR维护逻辑
class Partition {
    void checkISRUpdate() {
        // 1. 检查follower延迟
        for (Replica follower : AR) {
            if (inISR(follower)) {
                if (isLagging(follower)) {
                    moveToOSR(follower);
                }
            } else {
                if (!isLagging(follower)) {
                    moveToISR(follower);
                }
            }
        }
        
        // 2. 持久化ISR变更
        if (isrChanged) {
            updateZkIsrChange();
        }
    }
}
```

5. **与可用性的关系**
```plaintext
acks配置影响：

acks=all (-1)：
- 等待ISR中所有副本确认
- 最高可靠性
- 较低吞吐量

acks=1：
- 仅等待leader确认
- 中等可靠性
- 中等吞吐量

acks=0：
- 不等待确认
- 最低可靠性
- 最高吞吐量
```

6. **监控指标**
```plaintext
关键监控项：
1. UnderReplicatedPartitions
   - 表示副本同步滞后的分区数
   - 反映系统健康状况

2. IsrExpandsPerSec/IsrShrinksPerSec
   - ISR集合扩大/收缩的频率
   - 反映系统稳定性

3. ReplicaMaxLag
   - follower落后leader的最大消息数
   - 反映副本同步状况
``` 