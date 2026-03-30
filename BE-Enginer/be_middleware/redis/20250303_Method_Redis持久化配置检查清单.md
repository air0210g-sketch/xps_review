---
type: Method_Note
tags: [redis, 持久化, 检查清单, 运维]
links:
  - [[20250303_00_Redis核心技术与实战_结构笔记]]
  - [[20250303_Atomic_持久化RDB与AOF]]
  - [[20250303_Redis核心技术与实战_索引]]
date: 2025-03-03
---

# Redis 持久化配置检查清单

## 🆔 Identifier
ID: 20250303_Redis持久化配置
Status: DRAFT

## 🎯 Purpose & Value

*   **Pain point**：未显式配置持久化时，默认仅 RDB 或仅 AOF，易出现「以为有持久化却重启丢数」或「AOF 未 fsync 丢盘」。
*   **Use case**：新实例上线前、故障复盘时核对持久化与恢复链路。
*   **Expected output**：一份勾选完毕的清单 + 明确选型（RDB / AOF / 混合）及对应参数。

## ⚙️ Prerequisites

*   [ ] 已确定业务对「可接受丢失窗口」的要求（如：≤1 秒 / ≤1 分钟 / 可接受重启丢）。
*   [ ] 有 redis.conf 或等效配置的写权限（或通过运维流程变更）。

## 📝 Actionable Steps

### Step 1：确定持久化组合

| Step | What | How | Why |
|------|------|-----|-----|
| 1 | 选定主持久化方式 | 能接受分钟级丢失 → 仅 RDB；要尽量少丢（秒级）→ 开 AOF；既要恢复快又要少丢 → 开混合（aof-use-rdb-preamble yes） | 与业务 SLA 一致 |
| 2 | 若用 RDB，设定触发条件 | 在 redis.conf 中设置 save 行（如 `save 900 1` 表示 900 秒内至少 1 次写则 bgsave） | 控制快照频率与丢数据上界 |

### Step 2：AOF 与 fsync

| Step | What | How | Why |
|------|------|-----|-----|
| 1 | 设定 AOF 开关与 fsync 策略 | `appendonly yes`；`appendfsync everysec`（推荐）或 `always`（延迟高）/ `no`（交内核，丢数更多） | everysec 在延迟与丢失之间折中 |
| 2 | 若启用混合持久化 | `aof-use-rdb-preamble yes` | 恢复时先载入 RDB 再重放 AOF，加快恢复并减少丢失 |

### Step 3：确认持久化文件与目录

| Step | What | How | Why |
|------|------|-----|-----|
| 1 | 确认 dir 与文件名 | `dir /data/redis`；RDB 默认 dump.rdb，AOF 默认 appendonly.aof | 确保磁盘空间充足且备份/监控覆盖该路径 |
| 2 | 确认进程用户对该 dir 有写权限 | `ls -la /data/redis` 并检查运行 Redis 的用户 | 避免写失败导致持久化静默失效 |

## ✅ Acceptance Checklist

*   [ ] 已明确「RDB / AOF / 混合」中哪一种（或组合）。
*   [ ] redis.conf 中 save / appendonly / appendfsync / aof-use-rdb-preamble / dir 已按上表设定并生效。
*   [ ] 重启一次实例并触发一次写，确认对应 RDB 或 AOF 文件生成且可被再次加载。

## 🧠 Mechanism & Leverage

*   **Core mechanism**：RDB 用 fork+COW 做快照不阻塞主线程；AOF 用追加写 + fsync 控制落盘时机；混合用 RDB 做基准确认 + AOF 增量，兼顾恢复速度与丢失量。
*   **80/20**：先定「能丢多少」再选 appendfsync 和是否开 AOF，多数问题出在「未开 AOF 或 fsync=no 却以为数据已落盘」。

## 🧪 Next Step Experiment (MVE)

*   **Goal**：验证当前实例持久化配置与恢复。
*   **Action**：在测试环境执行：写一条 key → 执行 BGSAVE 或等待 AOF 落盘 → kill 进程 → 用当前 dir 下 RDB/AOF 启动 → 检查 key 是否存在。
*   **Expected result**：key 仍在；若不存在则回溯 save/appendfsync 与进程用户权限。
