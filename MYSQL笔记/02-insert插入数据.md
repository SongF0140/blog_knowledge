# 第2章 INSERT 插入数据

## 2.1 INSERT 的基本语法

`INSERT` 是向表中新增记录的命令，属于 DML（数据操纵语言）。最基本的语法是：

```sql
INSERT INTO 表名 (字段1, 字段2, ...) VALUES (值1, 值2, ...);
```

字段列表和值列表必须一一对应：字段个数要一致、顺序要一致、类型要匹配。例如向 `student` 表插入一条记录：

```sql
INSERT INTO student (name, gender, age, class_id, email)
VALUES ('张三', '男', 18, 1, 'zhangsan@example.com');
```

这里没有写 `id`，因为 `id` 是自增主键，MySQL 会自动分配。插入后可以查询验证结果。

## 2.2 省略字段列表的写法

如果插入的值严格按照建表时字段的顺序提供，且覆盖所有字段，可以省略字段列表：

```sql
INSERT INTO student VALUES (NULL, '李四', '女', 19, 2, 'lisi@example.com');
```

`NULL` 用于自增主键，表示让数据库自动生成。但这种写法依赖表结构的顺序，一旦表结构改动就容易出错，建议始终明确写出字段列表，可读性更高、更安全。

## 2.3 一次插入多行

MySQL 支持在一条 INSERT 中批量插入多行，用逗号分隔多个值列表，效率比逐条插入高：

```sql
INSERT INTO student (name, gender, age, class_id, email) VALUES
    ('王五', '男', 20, 1, 'wangwu@example.com'),
    ('赵六', '女', 21, 2, 'zhaoliu@example.com'),
    ('孙七', '男', 22, 3, 'sunqi@example.com');
```

## 2.4 INSERT ... SELECT 复制数据

`INSERT ... SELECT` 可以把一张表的查询结果插入到另一张表，常用于数据迁移或备份：

```sql
INSERT INTO student_backup (name, gender, age)
SELECT name, gender, age FROM student WHERE age >= 18;
```

这条语句先从 `student` 中查出年龄大于等于 18 的记录，再把结果写入 `student_backup`。需要注意两张表对应字段的类型要兼容。

## 2.5 插入流程与注意事项

```mermaid
flowchart LR
    A[准备字段与值] --> B{是否指定字段?}
    B -- 是 --> C[按字段列表匹配值]
    B -- 否 --> D[按建表顺序匹配值]
    C --> E[校验类型与约束]
    D --> E
    E --> F[写入表中]
    F --> G[自增主键自动生成]
```

插入时最容易踩的坑有三个：一是字符串和日期要用单引号包裹；二是不能向自增主键、带默认值的字段随意传非法值；三是违反非空、唯一、外键等约束时会直接报错。例如向 `name` 传 `NULL` 会触发 `NOT NULL` 约束失败。

## 2.6 常见问题 Q&A

**Q1：INSERT 时字段列表可以省略吗？**
A：可以，但必须按建表字段顺序提供所有值，自增主键通常用 NULL 占位。为了可读性和稳定，建议始终显式写出字段列表。

**Q2：一次插入多行有什么好处？**
A：一条 INSERT 插入多行只需一次往返和一次解析，比逐条插入效率高很多，适合批量导入数据。

**Q3：插入失败通常是什么原因？**
A：常见原因有：字段与值的数量或顺序不匹配、类型不兼容、违反 NOT NULL/UNIQUE/外键等约束、字符串忘了加引号。
