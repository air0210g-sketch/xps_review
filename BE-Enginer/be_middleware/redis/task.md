---
type: task-tracking
source: 146-Redis核心技术与实战.epub
tags: [redis, task]
date: 2025-03-03
---

# 《Redis核心技术与实战》学习任务追踪

## Preparation

- [x] Pre-game Plan Created（≥6 项 TODO + Context；见 `20250303_01_Redis核心技术与实战_执行计划.md`）

## Structure & Index（阶段一 + deep-learn Phase 1–2）

- [x] 阶段一：金字塔建模（全书知识图谱）已产出 → `20250303_00_Redis核心技术与实战_结构笔记.md`
- [x] Structure Note Created (Adler)（同上）
- [x] Index Note Created (Luhmann) → `20250303_Redis核心技术与实战_索引.md`
- [x] Index Note Onboarding (Phase 2.5)：挂载到 [[INDEX_后端中间件]]；无 03_索引，索引保留于本目录

## 递归生长 (Phase 3) & 阶段二

- [x] 首个核心难点 3W1H 深度解剖（单线程模型）→ `20250303_02_Redis单线程模型_3W1H解剖.md`
- [x] **[[20250303_02_Redis单线程模型_3W1H解剖]]**  
  *Luhmann Scan*: 前置 → Round 2: [[20250303_Atomic_IO多路复用]]. 连接 → Round 3: epoll/select、事件循环. 方法论 → （无；Phase 4 可补「避免慢命令」清单）
- [x] **[[20250303_Atomic_IO多路复用]]**  
  *Luhmann Scan*: 前置 → Round 2: 文件描述符、阻塞/非阻塞. 连接 → Round 3: [[20250303_02_Redis单线程模型_3W1H解剖]]、Nginx 事件模型. 方法论 → （无）
- [x] **[[20250303_Atomic_持久化RDB与AOF]]**  
  *Luhmann Scan*: 前置 → Round 2: fork、fsync. 连接 → Round 3: 主从复制、混合持久化. 方法论 → [[Redis持久化配置检查清单]] (Phase 4)

### Phase 3 Round 2（新增原子笔记 + Luhmann Scan）

- [x] **[[20250303_Atomic_主从复制]]**  
  *Luhmann Scan*: 前置 → Round 2: [[20250303_Atomic_持久化RDB与AOF]]. 连接 → Round 3: [[20250303_Atomic_哨兵]]、repl_backlog. 方法论 → （无）
- [x] **[[20250303_Atomic_哨兵]]**  
  *Luhmann Scan*: 前置 → Round 2: [[20250303_Atomic_主从复制]]. 连接 → Round 3: 集群、Raft. 方法论 → （无）
- [x] **[[20250303_Atomic_集群分片与路由]]**  
  *Luhmann Scan*: 前置 → Round 2: [[20250303_02_Redis单线程模型_3W1H解剖]]. 连接 → Round 3: hash tag、Gossip. 方法论 → （无）
- [x] **[[20250303_Atomic_分布式锁]]**  
  *Luhmann Scan*: 前置 → Round 2: SET NX PX、Lua. 连接 → Round 3: [[20250303_Atomic_Cache Aside缓存模式]]. 方法论 → [[20250303_Method_分布式锁实现检查清单]] (Phase 4)
- [x] **[[20250303_Atomic_Cache Aside缓存模式]]**  
  *Luhmann Scan*: 前置 → Round 2: 读穿/写穿. 连接 → Round 3: [[20250303_Atomic_分布式锁]]、布隆过滤器. 方法论 → （无）

## Methodology (Phase 4)

- [x] [[20250303_Method_Redis持久化配置检查清单]]（SOP/Checklist/MVE）
- [x] [[20250303_Method_避免慢命令清单]]（禁止 KEYS、慢日志、大 key）
- [x] [[20250303_Method_分布式锁实现检查清单]]（加锁/释放/续期、Lua）

## Review (Phase 5 & 6)

- [x] Feynman Check（单线程已含费曼检验；其余可后续补比喻）
- [x] Network Check（每张笔记 ≥2 链接；已入 INDEX_后端中间件；多索引见审查报告）

## Workflow Audit (Phase 6.5)

- [x] 流程执行审查（无 workflow-audit skill，已自检）→ `20250303_Redis核心技术与实战_流程审查报告_德明与葛文德视角.md`；DoD 全部通过
