---
type: method-note
tags: [MySQL, 主备, SOP, 延迟, binlog]
links: [[主备复制]], [[binlog]], [[binlog 格式]]
---

# 主备搭建与延迟排查 SOP

**目的**：搭建一主一备并验证同步；出现备库延迟时按步骤定位是 IO 线程、SQL 线程还是大事务/锁导致。

**机制**：主库写 binlog，备库 IO 线程拉取、SQL 线程重放。延迟 = 主已提交但备未重放到该位点；常见原因包括单线程重放阻塞、大事务、从库负载高、网络或磁盘 IO。

---

## 搭建主备（可执行步骤）

1. 主库：`my.cnf` 设 `log_bin=1`、`server_id=主库唯一值`；执行 `GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%'`；记下 `SHOW MASTER STATUS` 的 file 与 position。
2. 备库：`my.cnf` 设 `server_id=备库唯一值`（与主不同）；`CHANGE MASTER TO MASTER_HOST=..., MASTER_USER='repl', MASTER_PASSWORD='...', MASTER_LOG_FILE='上一步file', MASTER_LOG_POS=上一步position`；执行 `START SLAVE`。
3. 验证：`SHOW SLAVE STATUS\G` 中 `Slave_IO_Running`、`Slave_SQL_Running` 均为 Yes；在主库插入一行，备库能查到即同步正常。

---

## 延迟排查步骤

1. 看延迟秒数：`SHOW SLAVE STATUS\G` 中 `Seconds_Behind_Master`；若持续增大则确认为延迟。
2. 区分 IO / SQL：`Slave_IO_Running=No` 则先查网络、主库 binlog 是否可读、账号权限；`Slave_SQL_Running=No` 则看 `Last_SQL_Error`，常见为主键冲突或 DDL 顺序问题。
3. 两线程均为 Yes 但延迟大：查主库是否有大事务（长时间未提交）、备库是否单线程重放堵在某一 event；用 `SHOW PROCESSLIST` 看备库 SQL 线程当前执行的语句。

---

## 检查清单

- [ ] 主库 `log_bin`、`server_id` 已设；备库 `server_id` 与主不同。
- [ ] 主库已建复制账号并授权；备库 `CHANGE MASTER` 的 file/position 与主库当前一致（或从备份恢复后的位点）。
- [ ] `SHOW SLAVE STATUS` 无 `Last_IO_Error` / `Last_SQL_Error`；两 Running 为 Yes。
- [ ] 延迟时已区分 IO 线程问题 vs SQL 线程问题 vs 大事务/负载。

---

## MVE

在本机用 Docker 或两台实例建一主一备，主库执行 `INSERT` 后 5 秒内在备库 `SELECT` 能查到；再在主库执行一个 sleep(60) 的大事务，观察备库 `Seconds_Behind_Master` 是否先升后降。
