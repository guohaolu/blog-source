# 长期激励系统的事件驱动架构设计

1. **领域事件模型设计**
```java
// 1. 资格变更事件
@Data
public class QualificationChangedEvent {
    private String eventId;          // 事件ID
    private String employeeId;       // 员工ID
    private String planId;           // 激励方案ID
    private QualificationStatus status; // 资格状态
    private LocalDateTime changeTime;   // 变更时间
    private Map<String, Object> details;// 变更详情
}

// 2. 评议完成事件
@Data
public class ReviewCompletedEvent {
    private String eventId;
    private String employeeId;
    private String reviewId;
    private ReviewResult result;
    private List<ReviewComment> comments;
}

// 3. 合同签署事件
@Data
public class ContractSignedEvent {
    private String eventId;
    private String employeeId;
    private String contractId;
    private ContractStatus status;
    private LocalDateTime signTime;
}
```

2. **Topic设计与分区策略**
```plaintext
Topic设计：
├── incentive.qualification.changed.v1  # 资格变更事件
├── incentive.review.completed.v1      # 评议完成事件
├── incentive.contract.signed.v1       # 合同签署事件
└── incentive.payment.initiated.v1     # 资金发放事件

分区策略：
- 按激励方案类型分区
- 确保同一方案的事件顺序性
- 支持并行处理不同方案
```

3. **事件驱动流程**
```java
@Service
public class QualificationService {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Transactional
    public void updateQualification(QualificationDTO dto) {
        // 1. 更新资格状态
        qualificationRepository.update(dto);
        
        // 2. 发送资格变更事件
        QualificationChangedEvent event = new QualificationChangedEvent();
        // ... 设置事件属性
        
        // 3. 使用方案ID作为key，确保分区顺序
        kafkaTemplate.send("incentive.qualification.changed.v1", 
                         dto.getPlanId(), 
                         event);
    }
}

@Service
public class ReviewService {
    @KafkaListener(
        topics = "incentive.qualification.changed.v1",
        groupId = "review-service-group"
    )
    public void handleQualificationChanged(QualificationChangedEvent event) {
        // 1. 幂等性检查
        if (eventProcessed(event.getEventId())) {
            return;
        }
        
        // 2. 创建评议任务
        ReviewTask task = createReviewTask(event);
        
        // 3. 发送评议创建事件
        kafkaTemplate.send("incentive.review.created.v1", 
                         event.getPlanId(), 
                         new ReviewCreatedEvent(task));
    }
}
```

4. **事件驱动架构价值**
```plaintext
1. 业务解耦
   - 服务间通过事件异步通信
   - 降低系统耦合度
   - 便于服务独立扩展

2. 数据一致性
   - 最终一致性保证
   - 通过事件溯源恢复状态
   - 完整的业务链路追踪

3. 性能提升
   - 异步处理提高响应速度
   - 削峰填谷
   - 支持水平扩展

4. 业务灵活性
   - 新增功能只需订阅事件
   - 便于实现业务监控
   - 支持事件回放和重试
```

5. **可靠性保证**
```java
@Configuration
public class KafkaConfig {
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        // 可靠性配置
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }
} 
```