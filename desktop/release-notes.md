SQL LAB 1.2.0 · 字段选项与真实 CHECK 约束。

- 支持列级、表级 CHECK、CONSTRAINT 命名及 ENFORCED / NOT ENFORCED。
- INSERT、UPDATE、ALTER 校验条件；违反约束报 3819，整条语句回退。
- 支持 ALTER ADD / DROP CHECK，新增时检查已有数据。
- 支持 COMMENT 字段注释与 SHOW CREATE TABLE，JSON / SQL / IndexedDB 保留约束和注释。
- 生成列明确报未支持，避免被当成普通字段静默接受。
- 保留全部字段类型、初代补全与 45 道练习。

同一字段内多个 CHECK 请改写为多个表级约束；ALTER CHECK 切换、子查询/聚合及未列明函数尚不支持。升级前导出完整 JSON，旧版发布保留。真实 MySQL 8.4 差分和 iPhone Safari 实机尚未验证。详见 docs/field-options.md。
