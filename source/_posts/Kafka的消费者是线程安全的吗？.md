---
layout: post
title: Kafka的消费者是线程安全的吗？
date: 2024-03-19 21:43:47
categories:
  - 中间件
tags:
  - Kafka
---

Kafka的消费者（KafkaConsumer）不是线程安全的，具体表现在：

1. **官方说明**([1](https://kafka.apache.org/documentation/))
- KafkaConsumer不是线程安全的
- 所有网络I/O操作都发生在进行调用的线程中
- 可以安全地关闭消费者或者从另一个线程唤醒轮询

2. **正确使用方式**
- 单线程消费：一个消费者实例对应一个线程
- 多线程处理：消费单线程，处理多线程
- 多消费者实例：每个线程一个独立的消费者实例

3. **错误使用示例**
```java
// 错误示例：多线程共享一个消费者实例
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
ExecutorService executor = Executors.newFixedThreadPool(2);
executor.submit(() -> consumer.poll(Duration.ofMillis(100)));  // 线程1
executor.submit(() -> consumer.poll(Duration.ofMillis(100)));  // 线程2
```

4. **正确使用示例**
```java
// 正确示例1：每个线程一个消费者实例
ExecutorService executor = Executors.newFixedThreadPool(2);
executor.submit(() -> {
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        // 处理消息
    }
});

// 正确示例2：消费单线程，处理多线程
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
ExecutorService executor = Executors.newFixedThreadPool(2);
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        executor.submit(() -> processRecord(record));  // 异步处理
    }
}
```

5. **最佳实践建议**
- 使用消费者组机制实现并行消费
- 处理逻辑放入线程池异步执行
- 注意消费位移的正确提交
- 合理设置消费者数量和分区数