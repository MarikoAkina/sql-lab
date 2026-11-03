# 字段选项与 CHECK（1.2.0）

本版在 1.1.0 字段类型基础上增加真实 CHECK 校验和 COMMENT 字段注释，保留原有补全、45 道练习及进度格式。基础选项 NULL / NOT NULL、单列 PRIMARY KEY / UNIQUE、常量 DEFAULT、整数主键 AUTO_INCREMENT 继续支持。

## 可直接执行的费用示例

在工作台选择任一数据库，执行：

```sql
CREATE TABLE fee_check_demo (
  id INT PRIMARY KEY AUTO_INCREMENT,
  category VARCHAR(20) NOT NULL COMMENT '费用类别',
  amount DECIMAL(10,2) NOT NULL DEFAULT 0 COMMENT '费用金额',
  paid DECIMAL(10,2) DEFAULT 0,
  CONSTRAINT fee_amount_nonnegative CHECK (amount >= 0),
  CONSTRAINT fee_paid_range CHECK (paid >= 0 AND paid <= amount)
);
INSERT INTO fee_check_demo(category, amount, paid)
VALUES ('教材', 120.50, 100), ('交通', 30, 30);
SELECT * FROM fee_check_demo;
SHOW CREATE TABLE fee_check_demo;
```

下面的写入会报 3819，原有两行保持不变：

```sql
INSERT INTO fee_check_demo(category, amount) VALUES ('错误示例', -1);
UPDATE fee_check_demo SET paid = amount + 1;
```

## 支持范围

- 列级：`age INT CHECK (age BETWEEN 0 AND 150)`；可加 `CONSTRAINT 名称`。列级条件只能引用本列，跨列条件请用表级形式。
- 表级：`CONSTRAINT 名称 CHECK (paid <= amount)`；可省略名称，自动生成 `表名_chk_序号`。名称在同一数据库内不区分大小写且不得重复。
- 可设置 `ENFORCED`（默认）或 `NOT ENFORCED`。后者保留定义但不限制写入，仍校验定义的字段引用和表达式支持范围。
- `ALTER TABLE 表 ADD CONSTRAINT 名称 CHECK (...)` / `ADD CHECK (...)`：检查全部已有数据，失败整条回退。
- `ALTER TABLE 表 DROP CHECK 名称` / `DROP CONSTRAINT 名称`：本版仅用于 CHECK，不能用来删除主键或 UNIQUE。
- 新增列时可携带 CHECK；修改类型会重新验证所有约束。MODIFY 不写 CHECK 时保留原有 CHECK；再次指定同名 CHECK 会报重复，请先显式 DROP。
- 删除列时移除该列的列级 CHECK。被表级 CHECK 引用的列需先删除对应约束；引用检测保守，表达式中的同名标识也可能要求先 DROP。
- COMMENT 使用单/双引号字符串常量，最多 1024 个 Unicode 字符；支持空注释、中文和引号转义。MODIFY 未重写 COMMENT 时按完整列定义清除注释。
- `SHOW CREATE TABLE 表` 返回当前结构、注释及约束；DESC 保持原有六列。
- CHECK 和 COMMENT 随 IndexedDB、JSON 完整备份、SQL 导出与导入保存，新增信息不要求进度迁移。旧版没有 CHECK 的正常备份可继续读取；包含新选项的备份不要导入旧应用，升级前保留完整 JSON。

## 条件判断与失败处理

CHECK 在字段类型转换后计算，只有结果为假才违反约束；NULL 产生 UNKNOWN 时允许通过。因此 `CHECK(amount >= 0)` 并不禁止 NULL，禁止空值需另加 NOT NULL。这与 MySQL 8.4 的基础规则相同。金额使用既有 Decimal 精确计算。

INSERT / UPDATE / ALTER 每条语句原子执行，失败恢复结构、数据及模拟自增计数。真实 InnoDB 失败时可能消耗自增值，模拟器会回退该值。多语句遇错停止，之前成功语句保留；脚本文件导入使用整份回退。无法定位到具体写入值的约束错误指向失败语句的首个 token。

## 限制

- 每表最多 100 个 CHECK，每个表达式最多 10,000 字符；已有执行时间、步数与数据量限制仍生效。
- CHECK 可用基础比较、逻辑、括号、算术、IN、BETWEEN、LIKE、IS NULL、CASE，以及以下确定性函数：ABS、CEIL / CEILING、FLOOR、MOD、ROUND、LENGTH、CHAR_LENGTH、UPPER、LOWER、CONCAT、IFNULL、COALESCE、NULLIF、IF、SUBSTRING / SUBSTR、LEFT、RIGHT、TRIM、REPLACE（其原有参数限制仍适用）。其他函数明确报不支持。
- 不支持 CHECK 内子查询、聚合、窗口函数、带表名前缀的字段以及引用 AUTO_INCREMENT 列。
- **同一字段定义内连续多个 CHECK 暂不支持**：上游解析器只保留最后一个，本版主动拒绝，避免遗漏。请改成多个表级约束。多列各自一个列级 CHECK 正常支持。
- `ALTER CHECK ... ENFORCED / NOT ENFORCED` 暂不支持，请 DROP 后 ADD；重新添加 ENFORCED 时会验证当前数据。
- 生成列、自动时间戳、字符集 / COLLATE、外键、复合主键、存储引擎等仍未实现，不伪装支持。未涉及本轮的合法语法也可能受上游解析器限制。
- 字符串比较、类型转换、除零结果、SQL 模式等沿用模拟器规则，不等同完整 MySQL 服务。SHOW CREATE 的格式和隐含选项也不与 MySQL 逐字相同。

规则依据：[MySQL 8.4 CHECK Constraints](https://dev.mysql.com/doc/refman/8.4/en/create-table-check-constraints.html)、[CREATE TABLE](https://dev.mysql.com/doc/refman/8.4/en/create-table.html)。已按文档制定并执行测试，未完成真实 MySQL 8.4 服务器差分验证；未完成 iPhone Safari 实机验证。
