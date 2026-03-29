---
name: sql-optimization
description: 'Use when reviewing SQL optimization, GORM queries, table schema, db relationships, Pluck, First, Scan, Find, and deep pagination in this repository.'
argument-hint: '审查某段 Go / SQL 代码'
user-invocable: true
disable-model-invocation: false
---

# SQL Optimization Review

用于审查本仓库里的 SQL、GORM 查询和分页写法，重点检查字段是否写错、表结构和关联关系是否匹配、以及是否存在高数据量分页性能问题。

## 何时使用
- 需要检查某段 SQL 或 GORM 查询是否正确。
- 需要去 db 目录和相关 model 中核对表结构、字段、主键、外键和关联关系。
- 需要判断 Pluck、First、Scan、Find 的使用是否符合返回值形态。
- 需要识别深分页、大 offset、全表扫描和不合理排序带来的性能风险。

## 工作流程
1. 先确认查询意图：这段代码是查单条、查列表、查单列、做聚合，还是做导出/报表。
2. 在 db 目录中定位目标表结构，并沿着关联关系找到相关表：
   - 优先查看 TableName、gorm tag、主键、索引、foreignKey、references 等定义。
   - 如果表关系不够明确，再回看 app/editor/model 或实际查询所在文件，确认 Join / Preload / Where 依赖的字段来源。
3. 核对 SQL 或 GORM 中用到的字段是否真实存在：
   - select / where / order / group / having / join 条件里的列名必须和表结构一致。
   - 别名必须能映射到目标 struct 字段或扫描目标。
   - 跨表字段必须确认属于正确的表，避免把关联表字段写成主表字段，或反过来。
4. 校验 GORM 方法语义是否正确：
   - Pluck：只适合单列提取，目标通常应是 slice 或可接收单列结果的变量。
   - First：只适合取单条记录；如果业务上要“最新一条”或“最早一条”，必须确认排序条件明确。
   - Scan：适合自定义投影、聚合结果或原始 SQL 结果；目标结构体字段必须和别名对应。
   - Find：适合列表结果；如果实际只需要一条记录，应该优先考虑 First 或更明确的条件。
5. 检查深分页优化：
   - 如果出现大 offset / limit 翻页，判断是否会随着页数升高而明显变慢。
   - 如果表上有自增 id 或稳定递增键，并且业务允许按顺序翻页，优先建议用基于最后一条数据 id 的 seek/keyset 分页。
   - 常见形式是用 id > last_id 或 id < last_id 搭配固定 order，而不是不断增大的 offset。
6. 输出结论时按“问题 -> 影响 -> 证据 -> 修复建议”给出，区分：
   - 明确错误：字段不存在、方法语义错误、映射不匹配、关联关系错误。
   - 潜在问题：深分页、排序不稳定、索引缺失、结果集过大。

## 判断标准
- 查询中出现的每个表名和字段名都能在 schema / model 中找到依据。
- Join / Preload / Where / Order 的字段归属正确，没有混用主表和关联表字段。
- 目标变量类型与结果形态匹配。
- 方法选择与返回值数量一致。
- 对大数据量分页问题给出是否需要优化的结论。

## 完成检查
- 已确认涉及表的结构和关联关系。
- 已确认 SQL / GORM 中字段、别名和条件没有写错。
- 已确认 Pluck、First、Scan、Find 的使用场景正确。
- 已确认是否存在需要改成 keyset 分页的深分页场景。
- 已给出可执行的修复建议或说明为什么当前写法可接受。