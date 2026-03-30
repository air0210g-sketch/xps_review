---
type: Atomic_Note
tags: [redis, 持久化, RDB, AOF, 可靠性]
links:
  - [[20250303_00_Redis核心技术与实战_结构笔记]]
  - [[20250303_Redis核心技术与实战_索引]]
  - [[20250303_02_Redis单线程模型_3W1H解剖]]
date: 2025-03-03
---

# 持久化：RDB 与 AOF

## Core Concept

**Redis 持久化**是在内存数据集之外，把数据落盘以便重启后恢复。两种主路径：**RDB（快照）** 与 **AOF（追加写日志）**；可同时开启，重启时 AOF 优先。

*例如：RDB 是某一时刻的全量快照文件；AOF 是顺序追加的「写命令」日志，重放即可恢复。*

## Mechanism

*   **RDB**：在指定时间点 fork 子进程，子进程遍历内存写 snapshot 到文件（如 dump.rdb）。优点：单文件、恢复快、对主线程影响小（COW）；缺点：两次快照之间会丢数据、fork 大实例时可能有短暂阻塞。
*   **AOF**：每次写命令（或每秒/每步）追加到 AOF 文件。通过 **fsync** 策略控制落盘频率：always / everysec / no。优点：丢数据少（everysec 约丢 1 秒）；缺点：文件大、恢复要重放、写放大。
*   **混合持久化**（Redis 4+）：RDB 做基准确认，增量用 AOF 格式追加，兼顾恢复速度与丢失量。

## Zettelkasten Context

这是 [[20250303_00_Redis核心技术与实战_结构笔记]] 维度一「数据与存储」中的核心组件；与「单线程」的关系是：RDB 用子进程避免阻塞主线程，AOF 的 fsync 若为 always 会拉高延迟。下游可接「主从复制」（从库加载 RDB/AOF）、「高可用」选型。

> [!NOTE] Further Exploration
> 选型清单：能接受分钟级丢失用 RDB；要尽量少丢用 AOF everysec；既要恢复快又要少丢用混合。Method Note 可做「Redis 持久化配置检查清单」(Phase 4)。
