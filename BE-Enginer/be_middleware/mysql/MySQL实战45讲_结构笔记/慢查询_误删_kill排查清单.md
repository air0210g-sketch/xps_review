---
type: method-note
tags: [MySQL, 排查, 慢查询, 误删, kill]
links: [[MySQL基础架构]], [[行锁]], [[binlog]], [[一条SQL的更新流程]]
---

# 慢查询 / 误删数据 / kill 不掉 排查清单

**目的**：遇到「SQL 很慢」「误删了数据」「kill 不掉语句」时，按固定顺序排查，避免遗漏或误操作。

**机制**：慢查询多与锁等待、全表扫描、大事务有关；误删恢复依赖 binlog 重放；kill 不掉常因语句处于等锁或内部状态，需从连接/事务/锁三个层面查。

---

## 慢查询排查步骤

1. 用 `SHOW PROCESSLIST` 或 performance_schema 找到长时间处于 `Query` 或 `Locked` 的会话与 SQL 文本。
2. 对其中一条执行 `EXPLAIN`，按 [[explain解读与脑补执行步骤]] 看是否全表扫描、是否用错索引。
3. 若状态为 `Locked` 或 `Waiting for ... lock`：用 `SHOW ENGINE INNODB STATUS` 的 LATEST DETECTED DEADLOCK / TRANSACTION 段看谁持锁、谁在等锁；结合 [[行锁]]、[[间隙锁]] 判断是否大事务或锁范围过大。
4. 根据结论：改 SQL/加索引、缩短事务、或调整并发顺序。

---

## 误删数据恢复步骤

1. 立即停止对该库的写入或业务，避免覆盖 binlog 或继续写入。
2. 取最近一次全量备份，恢复到临时实例；从备份结束的 binlog 位点开始，用 mysqlbinlog 重放 binlog 到误删前的某一秒（`--stop-datetime` 或 `--stop-position`）。
3. 从临时实例导出需要恢复的表或行，再导入线上（或通过主备切换将临时库替为主库，视架构而定）。

---

## kill 不掉语句排查步骤

1. 确认 `KILL <id>` 的 id 为 `SHOW PROCESSLIST` 中的正确 Id；执行后观察该连接是否在数秒内消失。
2. 若仍存在：该语句可能处于等锁或内部等待（如 InnoDB 刷脏、MDL 等）；查 `information_schema.innodb_trx` 与 `SHOW ENGINE INNODB STATUS` 看是否有未提交事务持锁。
3. 若为备库 SQL 线程或复制相关，需按主备 SOP 处理（如跳过错误或重建从库），不能简单 kill 应用连接。

---

## 检查清单

- [ ] 慢查询：已用 PROCESSLIST/explain 定位到具体 SQL 与 type/key/rows；已区分「扫太多行」与「等锁」。
- [ ] 误删：已停写、已取全量备份+binlog 位点、已用 mysqlbinlog 重放到误删前并验证临时库数据。
- [ ] kill 不掉：已确认 KILL 的 id 正确；已查是否有长事务/持锁；已知是否为复制线程。

---

## MVE

在测试库执行一条 `SELECT ... FOR UPDATE` 不提交，另会话对同一行 UPDATE 会阻塞；在第三会话 `SHOW PROCESSLIST` 与 `SHOW ENGINE INNODB STATUS` 观察锁等待；对第一会话执行 COMMIT 后观察第二会话是否继续。重复一次「误删」一行后从 binlog 重放恢复该行」的完整流程。
