# Kafka消息消费失败处理方案

1. **重试机制设计**
```java
@Configuration
public class KafkaConsumerConfig {
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        // 禁用自动提交
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // 从最早的消息开始消费
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }
}

@Service
public class ReviewService {
    @KafkaListener(
        topics = "incentive.qualification.changed.v1",
        groupId = "review-service-group"
    )
    @RetryableKafkaHandler(
        attempts = "3",
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void handleQualificationChanged(
            @Payload QualificationChangedEvent event,
            Acknowledgment ack) {
        try {
            // 1. 业务处理
            processEvent(event);
            
            // 2. 确认消息
            ack.acknowledge();
            
        } catch (Exception e) {
            // 3. 重试次数达到上限，进入死信队列
            if (!isRetryable(e)) {
                sendToDLQ(event);
                ack.acknowledge();
                return;
            }
            throw e; // 触发重试
        }
    }
}
```

2. **死信队列处理**
```java
// 1. 死信队列配置
@Bean
public TopicBuilder deadLetterTopic() {
    return TopicBuilder.name("incentive.dlq.v1")
           .partitions(3)
           .replicas(3)
           .build();
}

// 2. 死信消息处理服务
@Service
public class DeadLetterService {
    @KafkaListener(
        topics = "incentive.dlq.v1",
        groupId = "dlq-processor-group"
    )
    public void processDLQ(ConsumerRecord<String, String> record) {
        // 1. 记录错误日志
        log.error("Processing DLQ message: {}", record);
        
        // 2. 错误通知
        notifyOperators(record);
        
        // 3. 持久化到错误表
        saveToErrorTable(record);
    }
    
    // 4. 手动重试接口
    public void retryMessage(String messageId) {
        Message msg = errorRepository.find(messageId);
        kafkaTemplate.send(msg.getOriginalTopic(), 
                         msg.getKey(), 
                         msg.getPayload());
    }
}
```

3. **错误数据持久化**
```java
@Entity
@Table(name = "kafka_error_messages")
public class ErrorMessage {
    @Id
    private String messageId;
    private String topic;
    private String key;
    private String payload;
    private String errorMsg;
    private int retryCount;
    private LocalDateTime createTime;
    private MessageStatus status;
}

public enum MessageStatus {
    FAILED,      // 失败待处理
    RETRYING,    // 重试中
    RESOLVED     // 已解决
}
```

4. **监控告警机制**
```java
@Component
public class KafkaMonitor {
    // 1. 消费延迟监控
    @Scheduled(fixedRate = 60000)
    public void checkLag() {
        Map<TopicPartition, Long> lags = getLags();
        if (isLagTooHigh(lags)) {
            alertService.send("消费延迟过高");
        }
    }
    
    // 2. 错误率监控
    @Scheduled(fixedRate = 60000)
    public void checkErrorRate() {
        double errorRate = getErrorRate();
        if (errorRate > threshold) {
            alertService.send("错误率过高");
        }
    }
}
```

5. **补偿机制**
```java
@Service
public class CompensationService {
    // 1. 定时补偿
    @Scheduled(cron = "0 0 */1 * * ?")
    public void compensate() {
        List<ErrorMessage> errors = 
            errorRepository.findUnresolved();
        
        for (ErrorMessage error : errors) {
            // 根据业务类型选择补偿策略
            CompensationStrategy strategy = 
                getStrategy(error.getTopic());
            strategy.compensate(error);
        }
    }
    
    // 2. 手动补偿接口
    public void manualCompensate(String messageId) {
        ErrorMessage error = 
            errorRepository.find(messageId);
        retryMessage(error);
    }
} 
```