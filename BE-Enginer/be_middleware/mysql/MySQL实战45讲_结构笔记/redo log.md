---
type: atomic-note
tags: [MySQL, InnoDB, redo log, WAL, crash-safe]
links: [[一条SQL的更新流程]], [[binlog]], [[MySQL基础架构]]
---

# redo log

**定义**：InnoDB 引擎专有的 **物理日志**，记录「在某个数据页上做了什么修改」。采用 **WAL（Write-Ahead Logging）**：先写 redo 并更新内存，再在合适时机把脏页刷盘，从而把随机写盘转为顺序写 redo，提升更新性能；并因此具备 **crash-safe**（异常重启后已提交记录不丢）。

**机制**：
- redo log 固定大小、**循环写**（例如 4 个 1GB 文件）；有 write pos（当前写位置）与 checkpoint（待擦除位置）；擦除前先把对应修改刷回数据文件。write pos 追上 checkpoint 表示写满，须先推进 checkpoint 才能继续写。
- 专栏比喻：掌柜的「粉板」— 先记在粉板上并更新内存，打烊后再把粉板内容整理进账本（数据文件）；粉板写满则先擦掉一部分再记新账。
- 与 binlog 区别：redo 是 InnoDB 的、物理的、循环写；binlog 是 Server 的、逻辑的、追加写。

**语境**：为何 MySQL 能恢复到任意秒？因为 binlog 全量保留 + 重放；而 redo 保证本机崩溃后已提交事务不丢。两阶段提交则保证 redo 与 binlog 一致。  
来源: 专栏第 02 讲。
