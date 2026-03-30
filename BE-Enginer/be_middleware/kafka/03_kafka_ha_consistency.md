# Kafka 数据一致性：水位 (HW) 与 Leader Epoch

> “副本机制解决了‘有备份’的问题，而数据一致性机制解决了‘备份之间数据不一样’的问题。”

## 1. 核心术语 (Key Terminology)
在 Log 文件中，有两个关键的 Offset 指针：
- **LEO (Log End Offset)**: 日志末端位移。记录了该副本底层 Log 文件的下一条消息写入位置。
- **HW (High Watermark)**: 高水位。**消费者只能拉取到 HW 之前的消息**。

> **定义**: $HW = \min(LEO_{Leader}, LEO_{Follower\_in\_ISR})$
> 即：ISR 集合中所有副本都已同步的最小 LEO。

### 为什么需要 HW？
防止“未提交消息”被消费。
如果 Leader 刚写入一条消息（LEO+1），但还没同步给 Follower，此时 Leader 挂了。
- 如果没有 HW：消费者读到了这条消息。新 Leader 上任后（因为没同步到）没有这条消息。**结果：消息凭空消失（消费端数据回滚），违反了一致性**。
- 有了 HW：这条消息虽然在 Leader 磁盘上，但 HW 没推它，消费者读不到。直到 Follower 同步成功，HW 推进，消费者才能读。

## 2. 数据截断 (Truncation)
当 Follower 故障重启，或者新 Leader 上任时，为了保证数据一致性，必须进行**截断**：
1.  **Follower 重启**: 读取之前的 HW，将自己的日志**截断到 HW**，然后开始从 Leader 拉取。
2.  **Leader 切换**: 新 Leader 上任，Follower 必须将自己的日志**截断到和新 Leader 一致**。

## 3. 痛点：HW 的缺陷 (Leader Epoch 的诞生)
单纯依赖 HW 会导致数据丢失或副本不一致。
*场景：只有 A, B 两个副本。*
1.  Leader A 写入 `msg1`，LEO=1。Follower B 拉取成功，LEO=1。
2.  Leader A 更新 HW=1。**在 A 通知 B 更新 HW 之前，B 挂了**。
3.  B 重启。B 只有 `msg1`，但它记录的 HW=0（因为没收到更新）。
4.  B 根据 HW=0 进行**截断**。`msg1` 丢失。
5.  A 挂了。B 成为 Leader。A 重启成为 Follower。
6.  A 的 HW=1，B 的 HW=0。数据不一致。

## 4. 救世主：Leader Epoch
从 0.11 版本引入。每条消息不仅有 Offset，还有 Epoch (版本号)。
- **Leader Epoch Sequence**: (Epoch, StartOffset) 对。
- **截断逻辑变更**: Follower 重启或切主时，不再盲目用 HW 截断，而是发请求问 Leader：“你这个 Epoch 的 StartOffset 是多少？”。
    - 如果 Follower 的数据比 Leader 多，截断多余的。
    - 如果 Follower 仅仅是 HW 落后但数据没少，则**不进行没必要的截断**。

这彻底解决了由 HW 更新异步导致的数据丢失问题。
