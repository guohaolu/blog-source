---
layout: post
title: MySQL可重复读隔离级别的实现原理
date: 2024-03-20 10:30:00
categories:
  - 数据库
tags:
  - MySQL
  - 事务隔离级别
  - MVCC
  - 锁机制
---

## MySQL默认隔离级别

MySQL InnoDB存储引擎默认使用**可重复读**（REPEATABLE READ）隔离级别。该级别通过MVCC（多版本并发控制）和锁机制的配合来实现。

## MVCC实现原理

### 1. 版本链

每行记录都存在一个版本链： 
```ascii
┌────────────┐ ┌────────────┐ ┌────────────┐
│ 最新记录 │ ──> │ 历史记录1 │ ──> │ 历史记录2 │
└────────────┘ └────────────┘ └────────────┘
```

### 2. 重要字段

每条记录都包含以下系统字段：

- `DB_TRX_ID`：创建/最后修改该记录的事务ID
- `DB_ROLL_PTR`：回滚指针，指向上一个版本
- `DB_ROW_ID`：行ID（可选）

### 3. 快照读实现

在事务开始时，会创建一个快照（Read View），包含：

- `creator_trx_id`：创建该Read View的事务ID
- `m_ids`：活跃的事务ID列表
- `min_trx_id`：活跃事务中最小的事务ID
- `max_trx_id`：下一个将被分配的事务ID

快照读判断规则：
```ascii
                是否可见？
                    │
            ┌───────┴───────┐
            ▼               ▼
   trx_id < min_trx_id   trx_id >= max_trx_id
        (可见)               (不可见)
            │               │
            └───────┬───────┘
                    ▼
            trx_id ∈ m_ids？
            ┌────┴────┐
            ▼         ▼
           是        否
        (不可见)    (可见)
```

## 锁机制

### 1. 记录锁（Record Lock）

- 锁定单个索引记录
- 防止其他事务修改或删除

### 2. 间隙锁（Gap Lock）

- 锁定索引记录之间的间隙
- 防止其他事务在间隙插入记录

### 3. Next-Key Lock

- 记录锁和间隙锁的组合
- 可以防止幻读

## 可重复读的实现过程

1. **事务开始**：
   - 创建Read View
   - 记录当前活跃事务

2. **读操作**：
   - 快照读：通过MVCC实现
   - 当前读：使用锁机制

3. **写操作**：
   - 加Next-Key Lock
   - 创建新的版本记录

## 示例说明

```sql
-- 事务1
START TRANSACTION;
SELECT * FROM users WHERE id = 1;  -- 创建Read View
-- 其他事务修改数据
SELECT * FROM users WHERE id = 1;  -- 使用相同Read View，看到相同结果
COMMIT;

-- 事务2
START TRANSACTION;
UPDATE users SET name = 'new_name' WHERE id = 1;  -- 加锁，创建新版本
COMMIT;
```

## 总结

MySQL通过以下机制实现可重复读：

1. MVCC保证读操作的一致性：
   - 版本链保存历史记录
   - Read View确定可见性
   
2. 锁机制保证写操作的隔离性：
   - 记录锁防止并发修改
   - 间隙锁防止幻读

这种实现既保证了数据的一致性，又提供了较好的并发性能。