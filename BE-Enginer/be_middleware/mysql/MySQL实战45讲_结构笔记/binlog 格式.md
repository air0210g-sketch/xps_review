---
type: atomic-note
tags: [MySQL, binlog, statement, row, mixed]
links: [[binlog]], [[主备复制]]
---

# binlog 格式

**定义**：binlog 有三种格式——**statement**（记录 SQL 原文）、**row**（记录行变更）、**mixed**（混合，由 MySQL 选择）。格式影响主备一致性、恢复精度与日志大小。

**机制**：statement 下某些函数/行为可能主备不一致；row 记录每行变更，一致性好但体积大。主备一致、延迟、临时表不记日志等都与 binlog 格式有关。

**语境**：配置主备、分析主备不一致或延迟时，需确认 binlog 格式及影响。  
来源: 专栏第 24 讲及后续；案例保真：部分（未逐字核对原文）。
