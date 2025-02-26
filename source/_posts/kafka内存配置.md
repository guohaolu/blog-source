# Kafka服务器内存配置最佳实践

1. **16G内存的最佳分配**
```plaintext
建议分配：
┌────────────────────┐
│ 系统预留: 2G      │ <- 操作系统和其他进程使用
├────────────────────┤
│ Kafka堆内存: 6G   │ <- broker的JVM堆内存
├────────────────────┤
│ 页面缓存: 8G      │ <- 用于消息数据缓存
└────────────────────┘

Swap建议：
- 默认4G -> 调整为1-2G
- 设置vm.swappiness=1
```

2. **为什么要减少Swap**
```plaintext
原因：
1. Kafka性能严重依赖于页面缓存
2. 一旦发生Swap：
   - 页面缓存被换出到磁盘
   - 数据访问延迟从ns级变为ms级
   - 吞吐量显著下降
   - 可能引发消息积压
```

3. **性能对比**
```plaintext
场景：生产者写入1GB数据

正常配置（小Swap）：
- 写入延迟：~10ms
- 吞吐量：~100MB/s
- 页面缓存命中率：95%

大Swap配置：
- 写入延迟：可能超过100ms
- 吞吐量：可能降至10MB/s
- 频繁的页面换入换出
```

4. **配置建议**
```bash
# 1. 调整Swap大小
sudo swapoff -a
sudo dd if=/dev/zero of=/swapfile bs=1G count=2
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. 修改swappiness
echo "vm.swappiness=1" >> /etc/sysctl.conf
sysctl -p

# 3. 设置Kafka堆内存
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
```

5. **监控指标**
```bash
# 监控Swap使用
vmstat 1
free -m

关键指标：
- si (swap in) 应接近0
- so (swap out) 应接近0
- swap used 应保持稳定
```

6. **风险防范**
```plaintext
1. 内存使用监控
   - 设置内存使用率告警
   - 阈值建议：80%

2. Swap使用监控
   - 设置Swap使用告警
   - 一旦发生Swap及时处理

3. 性能指标监控
   - 生产者延迟
   - 消费者延迟
   - 页面缓存命中率
``` 