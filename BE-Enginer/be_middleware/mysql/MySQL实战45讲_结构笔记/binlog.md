---
type: atomic-note
tags: [MySQL, binlog, 归档, 主备复制, 恢复]
links: [[一条SQL的更新流程]], [[redo log]], [[主备复制]]
---

# binlog

**定义**：MySQL **Server 层** 的 **逻辑日志**，记录语句的原始逻辑（如「给 ID=2 的 c 字段加 1」）。所有引擎都可使用；**追加写**，写满一个文件再写下一个，不覆盖历史，用于归档、按时间点恢复与主备复制。

**机制**：
- 与 redo 的三点不同：① redo 是 InnoDB 特有，binlog 是 Server 层、所有引擎可用；② redo 是物理日志（某数据页的修改），binlog 是逻辑日志（语句或行变更）；③ redo 循环写、binlog 追加写。
- 恢复「半个月内任意一秒」：取最近一次全量备份恢复到临时库，再从备份时间点起重放 binlog 到目标秒；两阶段提交保证 redo 与 binlog 一致，恢复结果正确。

**语境**：主备同步、从库延迟、binlog 格式（statement/row/mixed）等都与 binlog 直接相关；误删表后恢复也依赖 binlog。  
来源: 专栏第 02 讲。
