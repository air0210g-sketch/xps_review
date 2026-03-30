# Kafka 交付语义：Exactly-Once Semantics (EOS)

> “在分布式系统中，‘不丢不重’曾被认为是不可能三角。Kafka 通过幂等生产者和事务机制打破了这一魔咒。”

## 1. 三种语义 (The Three Semantics)
- **At-Most-Once (最多一次)**: 发消息不确认。快，但会丢数据。（Ack=0）
- **At-Least-Once (至少一次)**: 发消息如果没收到 Ack 就重试。数据不丢，但可能重复。（Ack=-1, Retries>0）。**这是默认行为**。
- **Exactly-Once (精确一次)**: 即使 Producer 重试，Broker 也就只记一次。

## 2. 幂等生产者 (Idempotent Producer)
> *解决单分区、单会话内的数据重复问题*

只要设置 `enable.idempotence=true` (默认已开启)，Kafka 就会自动去重。

### 原理
Producer 启动时申请一个 **PID (Producer ID)**。
每条发往特定 Partition 的消息，还会带一个单调递增的 **Sequence Number (SeqNum)**。

Broker 在内存维护 `(PID, Partition) -> LastSeqNum` 映射：
- 如果收到 `SeqNum = LastSeqNum + 1`：接受。更新 LastSeqNum。
- 如果收到 `SeqNum <= LastSeqNum`：**判定为重复发送 (Duplicate)**。直接返回 Ack，但不写入 Log。
- 如果收到 `SeqNum > LastSeqNum + 1`：说明中间丢了消息，抛出 `OutOfOrderSequenceException`。

## 3. 事务 (Transactions)
> *解决跨分区、跨会话的原子写入问题*

这也是 `consume-process-produce` 模式（流处理）的核心。
我们要保证：从 Topic A 消费，处理后写入 Topic B 和 Topic C，这三步要么全成功，要么全失败。

### 核心机制
1.  **Transactional ID**: 用户需提供全局唯一的 ID，用于跨会话恢复。
2.  **Transaction Coordinator**: Broker 端组件，管理事务状态。
3.  **Control Messages**: 在数据流中插入特殊的 **Marker (COMMIT/ABORT)** 消息。

### 隔离级别 (isolation.level)
Producer 写入未提交事务消息时，数据**确实**已经落盘了。
只有设置了 `isolation.level=read_committed` 的消费者，才会：
1.  先缓存未提交消息。
2.  见到 `COMMIT` Marker 后才吐给用户。
3.  见到 `ABORT` Marker 则丢弃之前缓存的消息。

## 4. 总结
- **幂等性**: 免费午餐，默认开启。防止网络重试导致的重复。
- **事务**: 昂贵午餐，需显式开启。用于精准流处理（Kafka Streams 核心）。
