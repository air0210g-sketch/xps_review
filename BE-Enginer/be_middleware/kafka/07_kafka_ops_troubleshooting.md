# Kafka 生产实践：运维与故障排查

> “Kafka 运维的两大‘癌症’：无休止的 Rebalance 和 永远消不完的 Lag。”

## 1. Rebalance (重平衡) 专题
**定义**: 让 Consumer Group 内的消费者重新分配 Partition 的过程。
**代价**: **Stop-The-World (STW)**. 在 Rebalance 期间，所有消费者停止消费。

### 为什么会频繁 Rebalance？
除了正常的加减节点，最常见的原因是**误判死亡**：
1.  **心跳超时**: 消费者 GC 时间过长，导致后台心跳线程无法发送心跳。
    - *Broker 视角*: 没收到心跳 -> 认为你挂了 ->踢出 -> Rebalance。
    - *优化*: 调大 `session.timeout.ms` (e.g. 6s -> 10s)。
2.  **消费超时**: 单批数据处理太慢，超过 `max.poll.interval.ms`。
    - *Broker 视角*: 你虽然活着发心跳，但你看似卡死了（没发 Poll 请求） -> 踢出 -> Rebalance。
    - *优化*: 调大 `max.poll.interval.ms` 或减小 `max.poll.records`。

### 救命稻草：Static Membership (静态成员)
**场景**: 滚动发布（Rolling Restart）。重启消费者会导致两次 Rebalance（一次离组，一次入组）。
**配置**: `group.instance.id` (设置固定值，如 Pod Name)。
**效果**: Broker 会认得你。当你重启（短暂离开）时，Partition 依然保留给你，**不会触发 Rebalance**。直到超时（`session.timeout.ms` 默认可设大到几分钟）。

## 2. Consumer Lag (积压) 治理
**监控**: `kafka-consumer-groups.sh --describe --group <group>` 查看 `LAG` 列。

### 解决方案
1.  **扩容 (Scale out)**:
    - 前提: `Consumer Count < Partition Count`。
    - 动作: 增加 Consumer 实例。
2.  **增加单机并行度 (Parallel Consumer)**:
    - 场景: 也就是 Partition 数已经封顶了，加 Consumer 没用。
    - 动作: 在 Consumer 内部起线程池。
    - *风险*: 位移提交（Commit Offset）变得复杂，容易丢消息或重复消费。建议使用 Confluent Parallel Consumer SDK。
3.  **紧急排毒 (Emergency Purge)**:
    - 场景: 积压几亿条，反正也处理不完了，且数据可丢。
    - 动作: 重置 Offset 到 Latest (`--reset-offsets --to-latest`)。

## 3. 调优清单 (Tuning Checklist)

| 组件         | 参数                  | 建议值        | 作用                     |
| :----------- | :-------------------- | :------------ | :----------------------- |
| **Broker**   | `log.retention.hours` | 72 (3天)      | 磁盘不够时优先砍这个     |
|              | `num.network.threads` | 3 -> 9        | 提升网络请求并发处理能力 |
| **Producer** | `batch.size`          | 16KB -> 32KB+ | 提高批处理吞吐           |
|              | `linger.ms`           | 0 -> 5-100ms  | 允许增加一点延迟来换吞吐 |
|              | `compression.type`    | lz4 / zstd    | 必开压缩                 |
| **Consumer** | `fetch.min.bytes`     | 1B -> 1KB+    | 没攒够数据别烦 Broker    |
