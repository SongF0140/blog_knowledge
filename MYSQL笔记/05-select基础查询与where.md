# 第5章 SELECT 基础查询与 WHERE

## 5.1 SELECT 的基本语法

`SELECT` 是查询数据的命令，属于 DQL，也是使用频率最高的 SQL。基本结构：

```sql
SELECT 字段列表 FROM 表名 WHERE 条件;
```

查询所有字段用 `*`：

```sql
SELECT * FROM student;
```

只查部分字段（投影）：

```sql
SELECT name, age FROM student;
```

## 5.2 使用别名 AS

可以用 `AS` 给字段或表起别名，让结果更易读：

```sql
SELECT name AS 姓名, age AS 年龄 FROM student;
```

`AS` 可以省略，直接写 `SELECT name 姓名 FROM student;`。

## 5.3 去除重复 DISTINCT

`DISTINCT` 用于去掉查询结果里的重复值：

```sql
SELECT DISTINCT class_id FROM student;
```

它会返回 `class_id` 的所有不同取值。注意 `DISTINCT` 作用于其后的所有字段组合。

## 5.4 WHERE 条件过滤

WHERE 支持丰富的运算符来筛选数据。

比较运算符：

```sql
SELECT * FROM student WHERE age >= 18;
SELECT * FROM student WHERE age <> 20;   -- 不等于，也可用 !=
```

逻辑运算符 `AND`、`OR`、`NOT`：

```sql
SELECT * FROM student WHERE age >= 18 AND class_id = 1;
```

范围查询 `BETWEEN ... AND ...`（闭区间）：

```sql
SELECT * FROM student WHERE age BETWEEN 18 AND 22;
```

集合查询 `IN`：

```sql
SELECT * FROM student WHERE class_id IN (1, 2, 3);
```

模糊查询 `LIKE`，`%` 匹配任意多个字符，`_` 匹配单个字符：

```sql
SELECT * FROM student WHERE name LIKE '张%';      -- 姓张
SELECT * FROM student WHERE name LIKE '张_';       -- 姓张且名字为两个字
```

空值判断 `IS NULL` / `IS NOT NULL`：

```sql
SELECT * FROM student WHERE email IS NULL;
```

## 5.5 查询流程

```mermaid
flowchart LR
    A[SELECT 字段] --> B[FROM 指定表]
    B --> C[WHERE 过滤行]
    C --> D[计算投影列]
    D --> E[返回结果集]
```

## 5.6 常见问题 Q&A

**Q1：`*` 和指定字段哪个更好？**
A：明确列出所需字段更高效、可读性更好，还能避免表结构变化带来的影响；`*` 只适合快速查看。

**Q2：LIKE 的 `%` 和 `_` 有什么区别？**
A：`%` 匹配任意多个字符（含 0 个），`_` 只匹配单个字符。

**Q3：WHERE 里能用别名吗？**
A：一般不能，因为 WHERE 先于 SELECT 的别名计算执行；要对别名过滤可用 HAVING 或子查询。
