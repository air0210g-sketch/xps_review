# Kafka 高可用与可靠性：副本机制 (Replication & ISR)

> “Kafka 并不是通过‘强一致性协议’（如 Raft/Paxos）来同步数据副本的，而是使用了一种更适合高吞吐场景的 ISR 机制。”

## 1. 副本角色 (Replica Roles)
Partition 的所有副本中，只有一个是 **Leader**，其余都是 **Follower**。
- **Leader**: 处理所有的读写请求。
- **Follower**: 被动从 Leader 拉取数据 (Fetch)，唯一的任务就是保持与 Leader 一致。

## 2. ISR (In-Sync Replicas) 设计
这是 Kafka 最精妙的设计之一。它没有采用“所有节点确认才算成功”（太慢），也没有采用“过半节点确认”（Quorum, 写放大）。

**ISR 集合**: 动态维护的一组“跟得上 Leader”的副本集合。
- 默认包含 Leader 自己。
- 只有 ISR 中的 Follower 才有资格被选为新 Leader。

### 判定标准
- **`replica.lag.time.max.ms`**: 如果 Follower 在该时间内没有向 Leader 发送 Fetch 请求，或者没以此跟上最新数据，就会被剔除出 ISR。
- *注：旧版本还有按消息条数 lag 的配置，已被废弃，因为容易造成抖动。*

## 3. `acks` 参数与持久性 (Durability)
Producer 发送消息时，通过 `acks` 参数控制可靠性级别：

| `acks` 值    | 描述                                         | 可靠性                                 | 延迟      |
| :----------- | :------------------------------------------- | :------------------------------------- | :-------- |
| **0**        | 发后即忘。不等待 Broker 确认。               | 最低 (可能丢)                          | 最低 (快) |
| **1**        | Leader 写入本地 Log 即返回成功。             | 中等 (Leader 挂且未同步则丢)           | 中等      |
| **all (-1)** | **Leader + 所有 ISR 副本**都写入成功才返回。 | 最高 (配合 `min.insync.replicas` 不丢) | 最高      |

### 关键配置: `min.insync.replicas`
如果 `acks=all` 但 ISR 里只有 Leader 一个人，那等于 `acks=1`。
为了避免这种情况，必须设置 `min.insync.replicas > 1`（生产环境通常设为 2）。
> **规则**: 当 `ISR 数量 < min.insync.replicas` 时，Broker 会拒绝写入（抛出 `NotEnoughReplicasException`）。这是一种“宁可不可用，也不可丢数据”的保护机制。

## 4. Unclean Leader Election
如果 ISR 里的副本全挂了，剩下的都是 lag 严重的副本，怎么办？
- `unclean.leader.election.enable=false` (默认): 宁死不屈。集群不可用，直到 ISR 副本恢复。
- `unclean.leader.election.enable=true`: 矮子里拔将军。允许非 ISR 副本成为 Leader。**代价是数据丢失和不一致**。

## 5. 总结
Kafka 的 HA 是一个权衡游戏：
- 想不丢数据：`acks=all` + `min.insync.replicas >= 2`
- 想高可用：`unclean.leader.election.enable=true` (慎用)
