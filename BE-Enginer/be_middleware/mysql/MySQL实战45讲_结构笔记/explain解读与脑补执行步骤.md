---
type: method-note
tags: [MySQL, explain, 执行计划, 索引]
links: [[索引（B+树与InnoDB）]], [[MySQL基础架构]]
---

# explain 解读与「脑补」执行过程步骤

**目的**：用 `EXPLAIN` 结果判断是否走索引、是否回表、扫描行数是否过大，并能在脑中还原「执行器如何调引擎、引擎如何扫表/走索引」。

**机制**：优化器选定执行计划后，explain 展示的是该计划的预估：用哪张表、用哪个索引、扫描约多少行、是否用临时表/文件排序等。对照 [[索引（B+树与InnoDB）]] 可知走二级索引+回表 vs 覆盖索引的差异，从而判断是否需改索引或改 SQL。

---

## 可执行步骤

1. 在目标 SELECT 前加 `EXPLAIN`（或 `EXPLAIN FORMAT=TREE` / `EXPLAIN ANALYZE`，视版本而定），执行后取结果集。
2. 看 **type**：`const`/`eq_ref` 通常最优，`ref` 次之，`range` 为范围扫描，`index` 为全索引扫描，`ALL` 为全表扫描。若为 `ALL` 且表大，优先考虑加索引或改条件。
3. 看 **key**：实际使用的索引；若为 NULL 而 possible_keys 有值，说明优化器未选用索引，可尝试 `FORCE INDEX` 或改 SQL 验证。
4. 看 **rows**：预估扫描行数；结合 **Extra** 中 `Using index`（覆盖，无需回表）、`Using filesort`/`Using temporary`（可能需优化 order by / group by）。
5. 「脑补」执行：根据 type/key/rows，想象执行器先调引擎「取满足条件的第一行」，再循环「取下一行」，直到结束；若走二级索引，每次取到主键后回表取其余列。

---

## 检查清单

- [ ] 已对慢 SQL 执行 `EXPLAIN` 并保存结果。
- [ ] 已确认 type 非 `ALL`（或已知全表扫描原因）；key 为期望的索引。
- [ ] 已根据 rows 与表大小判断扫描量是否可接受；Extra 中是否有 filesort/temporary，是否可通过索引消除。

---

## MVE

对一张有主键和二级索引的表，分别执行「WHERE 主键=值」与「WHERE 二级索引列=值」的 SELECT，对比两次 EXPLAIN 的 key、type、rows，确认第二次是否走二级索引、是否出现回表（Extra 无 `Using index` 即回表）。
