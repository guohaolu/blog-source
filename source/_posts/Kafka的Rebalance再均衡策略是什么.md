---
layout: post
title: Kafka的Rebalance再均衡策略是什么？
date: 2024-03-19 21:43:47
categories:
  - 中间件
tags:
  - Kafka
---

1. **什么是Rebalance**([1](https://kafka.apache.org/documentation/))
- 当消费者组成员发生变化时触发的分区重新分配机制
- 目的是实现负载均衡，让分区尽可能均匀地分配给所有消费者

2. **触发条件**
- 消费者组成员数量发生变化（新增或减少消费者）
- 订阅的主题数量发生变化
- 主题的分区数发生变化
- 消费者宕机或网络故障

3. **再均衡策略**
- **Range策略（默认）**
  - 按照分区号范围进行分配
  - 可能会导致分配不均
  - 分配公式：分区号/消费者数量
  
- **RoundRobin策略**
  - 轮询分配方式
  - 分区分配更均匀
  - 适合分区数较多的场景

- **Sticky策略**
  - 分配尽可能与上次保持相同
  - 减少分区迁移带来的开销
  - 在出现故障时才进行必要的分区移动

4. **再均衡过程**
- 消费者组选举Group Coordinator
- Group Coordinator选举Leader Consumer
- Leader制定分配方案
- Group Coordinator将方案下发给所有消费者

5. **注意事项**
- 再均衡期间消费者无法消费消息
- 频繁的再均衡会影响系统性能
- 合理设置session.timeout.ms和heartbeat.interval.ms
- 建议使用Sticky策略减少不必要的分区移动