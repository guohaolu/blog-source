---
layout: post
title: Kafka的acks机制详解
date: 2024-03-19 21:43:47
categories:
  - 中间件
tags:
  - Kafka
  - 消息队列
  - 可靠性
---

本文详细介绍Kafka的acks机制及其实现原理。

1. **acks=0 (fire and forget)**
```plaintext
发送流程：
Producer -----> Leader
         不等待确认

特点：
- 生产者发完即忘
- 不等待任何确认
- 最大吞吐量

风险：
┌─────────┐    ┌─────────┐
│Producer │    │ Leader  │ ✗ (宕机)
└─────────┘    └─────────┘
     │             │
     └──消息───────┘
     (消息丢失)

适用场景：
- 日志收集
- 监控数据
- 允许少量丢失
```

2. **acks=1 (leader only)**
```plaintext
发送流程：
Producer -----> Leader -----> Follower
         等待Leader确认    异步复制

确认过程：
┌─────────┐    ┌─────────┐
│Producer │    │ Leader  │ ✓ (确认)
└─────────┘    └─────────┘
     │             │
     └──消息───────┘
     │             │
     └──ACK────────┘

风险场景：
Leader确认后立即宕机，数据未同步到Follower：
┌─────────┐    ┌─────────┐
│Producer │    │ Leader  │ ✓ -> ✗ (宕机)
└─────────┘    └─────────┘
                    │
              未同步 │
                    ↓
               ┌─────────┐
               │Follower │
               └─────────┘
```

3. **acks=-1/all (all in sync replicas)**
```plaintext
发送流程：
Producer -----> Leader -----> ISR所有副本
         等待所有ISR确认

确认过程：
┌─────────┐    ┌─────────┐
│Producer │    │ Leader  │ ✓
└─────────┘    └─────────┘
     │             │
     │        同步  │
     │             ↓
     │        ┌─────────┐
     │        │Follower │ ✓
     │        └─────────┘
     │             │
     └────ACK──────┘

配置建议：
min.insync.replicas=2
replication.factor=3
```

4. **性能对比**
```plaintext
延迟对比：
acks=0  < 1ms
acks=1  ~10ms
acks=-1 ~100ms

吞吐量对比(消息/秒)：
acks=0  100,000+
acks=1  50,000+
acks=-1 10,000+
```

5. **最佳实践**
```java
// 重要业务数据配置
Properties props = new Properties();
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

// 非重要数据配置
Properties props = new Properties();
props.put(ProducerConfig.ACKS_CONFIG, "1");
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
``` 