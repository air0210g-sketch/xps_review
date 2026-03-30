---
type: Atomic_Note
tags: [redis, 缓存, Cache Aside, 穿透击穿雪崩]
links:
  - [[20250303_00_Redis核心技术与实战_结构笔记]]
  - [[20250303_Redis核心技术与实战_索引]]
  - [[20250303_Atomic_分布式锁]]
date: 2025-03-03
---

# Cache Aside 缓存模式

## Core Concept

**Cache Aside** 是「应用自己管缓存」的模式：**读**时先查缓存，未命中再查 DB 并回填缓存；**写**时先写 DB，再删缓存（或更新缓存）；缓存不作为数据源，DB 为准，缓存是加速层。

*例如：读用户信息先 get user:123，无则查库并 set user:123；更新用户时先 update DB，再 del user:123，下次读时自然回填。*

## Mechanism

*   **读路径**：cache hit 直接返回；cache miss → 查 DB → 写回缓存（可设 TTL 防脏数据长期存在）。
*   **写路径**：先更新 DB，再**删除**缓存（而非更新缓存），避免并发写导致缓存与 DB 长期不一致；下次读时 cache miss 再回填。
*   **穿透/击穿/雪崩**：穿透 = 查不存在 key，请求直打 DB，用布隆过滤器或缓存空值缓解；击穿 = 热点 key 过期瞬间大量请求打 DB，用互斥锁（如 [[20250303_Atomic_分布式锁]]）或逻辑过期；雪崩 = 大量 key 同时过期，打散 TTL 或多级缓存。

## Zettelkasten Context

是 [[20250303_00_Redis核心技术与实战_结构笔记]] 维度三「缓存模式」的代表。与 [[20250303_Atomic_分布式锁]] 组合可解决击穿（单请求回源）。对比 Read/Write Through：Cache Aside 由应用控制读写；Through 由缓存层封装，对应用透明。

> [!NOTE] Further Exploration
> 延迟双删（先删缓存再更新 DB 再删缓存）的适用场景；与 DB 强一致的替代方案（如 Canal 订阅 binlog 更新缓存）。
