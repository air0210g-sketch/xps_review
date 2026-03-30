---
type: atomic-note
tags: [MySQL, 更新流程, redo log, binlog, 两阶段提交]
links: [[20250228_00_MySQL实战45讲_结构笔记]], [[MySQL基础架构]], [[redo log]], [[binlog]], [[两阶段提交]], [[崩溃恢复]]
---

# 一条SQL的更新流程

**定义**：更新语句与查询走同一套 Server 层（连接器→分析器→优化器→执行器），区别在于执行器与引擎配合写数据时，会涉及 **redo log**（InnoDB）与 **binlog**（Server），并通过 **两阶段提交** 保证两份日志逻辑一致，从而支持崩溃恢复与按时间点恢复。

**机制**（以 `update T set c=c+1 where ID=2` 为例）：
1. 执行器向引擎要 ID=2 的那一行；引擎从内存或磁盘读入后返回。
2. 执行器把 c 加 1，得到新行，再调引擎写回。
3. 引擎把新数据写入内存，并把本次更新写入 **redo log**，置为 **prepare**，再通知执行器完成、可提交。
4. 执行器写 **binlog** 并落盘。
5. 执行器调引擎提交事务；引擎把 redo log 从 prepare 改为 **commit**。

两阶段提交（redo prepare → 写 binlog → redo commit）保证：崩溃恢复时若 redo 有 prepare 无 commit，则用 binlog 是否写入决定是提交还是回滚，避免恢复后与用 binlog 做的备份/从库不一致。

**语境**：理解「为什么 MySQL 能恢复到半个月内任意一秒」：全量备份 + 该时间点之后的 binlog 重放；若没有两阶段提交，先写 redo 再写 binlog 或反之，都会导致恢复后与备份不一致。  
来源: 专栏第 02 讲。
