# 第8章 联表查询 JOIN 详解

## 8.1 为什么需要联表查询

数据被拆分到多张表后，要拿到完整信息就需要把多张表"拼"起来。例如想同时显示"学生姓名"和"所在班级名"，姓名在 `student` 表，班级名在 `class` 表，这时就要用 `JOIN` 联表查询。JOIN 的本质是按某个关联条件把两张表的行两两组合，再按类型决定保留哪些行。

## 8.2 内连接 INNER JOIN

内连接只返回两张表中"关联字段都匹配"的行，是最常用的连接：

```sql
SELECT s.name, c.name AS class_name
FROM student s
INNER JOIN class c ON s.class_id = c.id;
```

这里给表起了别名 `s`、`c`，`ON` 后面写关联条件。没有班级或班级不存在对应关系的学生会被排除。

## 8.3 左连接 LEFT JOIN

左连接返回左表的全部行，右表匹配不上时用 NULL 填充：

```sql
SELECT s.name, c.name AS class_name
FROM student s
LEFT JOIN class c ON s.class_id = c.id;
```

即使某个学生的 `class_id` 在 class 表中不存在，也会保留该学生，`class_name` 显示为 NULL。常用于"主表全部 + 关联表可选信息"的场景。

## 8.4 右连接 RIGHT JOIN

右连接与左连接相反，保留右表全部行：

```sql
SELECT s.name, c.name AS class_name
FROM student s
RIGHT JOIN class c ON s.class_id = c.id;
```

会返回所有班级，即使某班级没有学生。实际中左连接更常用，右连接往往可以改写成左连接。

## 8.5 各种连接的关系

```mermaid
flowchart TD
    A[INNER JOIN 交集] --> E
    B[LEFT JOIN 左全 + 交集] --> E
    C[RIGHT JOIN 右全 + 交集] --> E
    D[FULL JOIN 并集] --> E
    E[按 ON 条件组合两表行]
```

MySQL 不直接支持 `FULL OUTER JOIN`，可以用 `LEFT JOIN` 和 `RIGHT JOIN` 的 `UNION` 来模拟：

```sql
SELECT s.name, c.name FROM student s LEFT JOIN class c ON s.class_id = c.id
UNION
SELECT s.name, c.name FROM student s RIGHT JOIN class c ON s.class_id = c.id;
```

## 8.6 自连接与子查询

**自连接**是把同一张表当成两张表来用，典型场景是查"员工及其上级"：

```sql
SELECT e.name AS 员工, m.name AS 上级
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id;
```

**子查询**是在一个查询里嵌套另一个查询，常见于 IN、比较运算：

```sql
SELECT name FROM student
WHERE class_id IN (SELECT id FROM class WHERE name = '一班');
```

## 8.7 常见问题 Q&A

**Q1：ON 和 WHERE 有什么区别？**
A：ON 是连接条件，在生成连接结果时起作用；WHERE 是连接完成后再过滤。对 INNER JOIN 两者结果常相同，对 LEFT JOIN 如果把右表条件写进 WHERE 会把 NULL 行过滤掉，效果大不相同。

**Q2：多张表怎么连？**
A：可以连续写多个 JOIN，每次用 ON 指定与哪张表关联，注意别名不要混淆即可。

**Q3：JOIN 和子查询怎么选？**
A：需要同时展示多表字段时用 JOIN；只需要判断存在性或用聚合值过滤时，子查询（如 `IN`、`EXISTS`）往往更清晰。
