---
type: method-note
tags: [MySQL, SOP, 执行流程, 自检]
links: [[MySQL基础架构]], [[20250228_00_MySQL实战45讲_结构笔记]]
---

# SQL 执行流程自检清单

**目的**：排查「这条 SQL 为什么慢 / 报错」时，按执行阶段逐项排除，快速定位是连接、解析、优化选索引还是引擎执行的问题。

**机制**：SQL 必经连接器→（查询缓存，8.0 已无）→分析器→优化器→执行器→引擎。每一阶段对应一类故障；按顺序勾选可避免跳过环节或重复排查。

---

## 检查清单（按执行顺序勾选）

- [ ] **连接**：`SHOW PROCESSLIST` 是否有该连接；是否 `Access denied` 或连接被 `wait_timeout` 断开（默认 8 小时）。
- [ ] **权限**：报错是否含 `SELECT command denied` / `Unknown column` — 前者为执行器阶段权限，后者为分析器阶段列名错误。
- [ ] **语法**：是否 `You have an error in your SQL syntax` — 分析器词法/语法；看 `near '...'` 定位位置。
- [ ] **优化器**：是否用错索引 — 用 `EXPLAIN` 看 `key`、`rows`；必要时用 `FORCE INDEX` 或改 SQL 验证。
- [ ] **执行器 + 引擎**：`EXPLAIN` 中 `rows_examined` 是否过大；是否锁等待（`SHOW ENGINE INNODB STATUS` 或 performance_schema）。

---

## 最小可行性实验（MVE）

任选一条已知慢的 SELECT，按上表从连接开始勾到「执行器+引擎」，记录卡住的那一项；再根据该项查对应原子笔记（如优化器→[[索引（B+树与InnoDB）]]，锁→[[行锁]]）。
