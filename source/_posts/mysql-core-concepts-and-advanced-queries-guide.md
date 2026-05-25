---
title: 夯实后端基石：MySQL 核心约束、进阶查询与底层架构
date: 2026-05-24 11:30:00
tags: [MySQL, 数据库, SQL, 架构设计, 面试突击]
categories: [技术底层深度剖析, 数据库与中间件]
---

在构建后端系统时，MySQL 是最核心的存储引擎。深入理解其约束机制、查询语法及底层架构，是编写高效、安全数据库代码的必备技能。本文将系统梳理 MySQL 的核心知识点，从数据类型与约束，到 CRUD 进阶操作，再到多表查询与视图，帮助读者建立完整的 MySQL 知识体系。

---

## 一、 数据库与文件基础

数据库与普通文件的核心差异在于数据管理的效率、安全性和并发能力。

### 1. 为什么选择数据库而非文件存储

| 缺陷 | 文件存储 | 数据库 |
|------|----------|--------|
| 查询效率 | 需遍历文件 | 索引机制 $O(\log N)$ |
| 海量数据 | GB 级文件撑爆内存 | 分布式分片支持 |
| 数据安全 | 断电易损坏 | 事务 + 崩溃恢复 |
| 程序控制 | 需自行解析格式 | SQL 标准化接口 |

### 2. 字符集与校验集

* **字符集**：决定数据如何**存取**
* **校验集**：决定数据如何**比较和排序**
* **注意**：字符集和校验集必须对应匹配

---

## 二、 数据类型详解

### 1. 数值类型

* `BIT`、`BOOL`：位类型和布尔类型
* `FLOAT`：浮点数（有精度丢失风险）
* `DECIMAL`：高精度浮点数（金融场景首选）

### 2. 文本、二进制类型

* `CHAR(n)`：定长字符串，不足部分用空格填充
* `VARCHAR(n)`：变长字符串，只存储实际字符

### 3. 时间日期类型

| 类型 | 格式 | 占用空间 | 说明 |
|------|------|----------|------|
| `DATE` | `yyyy-mm-dd` | 3 字节 | 仅日期 |
| `DATETIME` | `yyyy-MM-dd hh:mm:ss` | 8 字节 | 范围 1000-9999 年 |
| `TIMESTAMP` | 同 DATETIME | 4 字节 | 自动更新的时间戳 |

### 4. 字符串类型

* `ENUM`：枚举，单选类型
* `SET`：集合，多选类型

---

## 三、 约束机制

约束是通过底层技术手段倒逼程序员，确保插入表中的数据正确且符合预期。

### 1. NULL 约束

```sql
CREATE TABLE user (
    name VARCHAR(20) NOT NULL  -- 该字段不能为空
);
```

### 2. 默认值约束

```sql
CREATE TABLE user (
    status INT DEFAULT 1  -- 用户未指定时使用默认值
);
```

### 3. 列描述

```sql
CREATE TABLE user (
    name VARCHAR(20) COMMENT '用户名'  -- 类似注释，用于字段说明
);
```

### 4. ZEROFILL 前导零填充

```sql
CREATE TABLE tmp (
    num INT(5) ZEROFILL  -- 不足5位时前面补零
);
```

### 5. 主键约束

* **唯一性**：主键值不能重复
* **非空性**：主键值不能为空
* **单表唯一**：一张表只能有一个主键

```sql
CREATE TABLE student (
    id INT PRIMARY KEY
);
```

**复合主键**：多个字段共同作为主键

```sql
CREATE TABLE score (
    stu_id INT,
    course_id INT,
    PRIMARY KEY (stu_id, course_id)  -- 两字段不能同时重复
);
```

### 6. 自增长约束

```sql
CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT  -- 自动从1开始递增
);
```

### 7. 唯一键约束

唯一键允许为空，空字段不做唯一性比较。

```sql
CREATE TABLE student (
    id INT PRIMARY KEY,
    telephone VARCHAR(32) UNIQUE KEY  -- 电话号唯一但可为空
);
```

### 8. 外键约束

外键用于定义主表和从表之间的关系。

```sql
CREATE TABLE IF NOT EXISTS student (
    id INT PRIMARY KEY,
    name VARCHAR(20),
    class_id INT,
    FOREIGN KEY (class_id) REFERENCES class(class_id)
);
```

**约束规则**：
* 从表增改：外键值必须在主表主键列中存在
* 从表删改：主表记录被从表引用时禁止删除
* 主表操作：受外键依赖关系限制

---

## 四、 CRUD 进阶操作

### 这里不做赘述

## 五、 分组与聚合

### 1. GROUP BY 分组

**按单字段分组**

```sql
SELECT deptno, MAX(sal) AS 最高, AVG(sal) AS 平均
FROM emp GROUP BY deptno;
```

**按多字段分组**

```sql
SELECT deptno, job, AVG(sal), MIN(sal)
FROM emp GROUP BY deptno, job;
```

### 2. HAVING 与 WHERE 的本质区别

| 特性 | WHERE | HAVING |
|------|-------|--------|
| 执行时机 | 分组/聚合之前 | 分组/聚合之后 |
| 筛选对象 | 原始数据行 | 聚合结果 |
| 支持聚合函数 | ❌ 不支持 | ✅ 支持 |

**案例**

```sql
SELECT deptno, job, AVG(sal) AS myavg
FROM emp
WHERE ename != 'SMITH'
GROUP BY deptno, job
HAVING myavg < 2000;
```

---

## 六、 内置函数

### 1. 聚合函数

```sql
SELECT COUNT(*) FROM exam_result;           -- 计数
SELECT SUM(math) FROM exam_result;          -- 求和
SELECT AVG(math) FROM exam_result;          -- 平均值
SELECT MAX(math) FROM exam_result;          -- 最大值
SELECT MIN(math) FROM exam_result;          -- 最小值
SELECT COUNT(DISTINCT math) FROM exam_result; -- 去重计数
```

### 2. 日期函数

```sql
SELECT DATE_ADD('2050-01-01', INTERVAL 10 DAY);  -- 增加天数
INSERT INTO tmp (birthday) VALUES (DATE(CURRENT_TIMESTAMP()));  -- 插入当前日期
```

### 3. 字符串函数

| 函数 | 说明 |
|------|------|
| `CONCAT(str1, str2)` | 拼接字符串 |
| `INSTR(str, substr)` | 查找子串位置 |
| `REPLACE(str, old, new)` | 替换子串 |
| `SUBSTRING(str, pos, len)` | 截取子串 |
| `LENGTH(str)` | 字节长度 |

### 4. 数学函数

| 函数 | 说明 |
|------|------|
| `ABS(n)` | 绝对值 |
| `CEILING(n)` | 向上取整 |
| `FLOOR(n)` | 向下取整 |
| `MOD(n, d)` | 取模 |
| `RAND()` | 随机数 [0,1) |

### 5. 其他函数

```sql
SELECT USER();                    -- 当前用户
SELECT DATABASE();                -- 当前数据库
SELECT MD5('admin');             -- MD5摘要
SELECT IFNULL(val1, val2);       -- NULL替代
```

---

## 七、 多表查询

### 1. 笛卡尔积

```sql
SELECT * FROM emp, dept;  -- 产生 m*n 条记录
```

### 2. 多表连接

**等值连接**

```sql
SELECT EMP.ename, EMP.sal, SALGRADE.grade
FROM EMP, SALGRADE
WHERE EMP.sal BETWEEN SALGRADE.losal AND SALGRADE.hisal;
```

### 3. 自连接

同一张表连接查询自身。

```sql
SELECT leader.empno, leader.ename
FROM emp leader, emp worker
WHERE leader.empno = worker.mgr
    AND worker.ename = 'FORD';
```

### 4. 子查询

**单行子查询**

```sql
SELECT ename, job
FROM EMP
WHERE sal = (SELECT MAX(sal) FROM EMP);

SELECT ename, sal
FROM EMP
WHERE sal > (SELECT AVG(sal) FROM EMP);
```

**多行子查询**

```sql
-- IN：匹配列表中任意值
SELECT * FROM emp
WHERE job IN (SELECT DISTINCT job FROM emp WHERE deptno = 10)
    AND deptno <> 10;

-- ALL：比所有值都大/小
SELECT * FROM EMP WHERE sal > ALL (SELECT sal FROM EMP WHERE deptno = 30);

-- ANY：比任意一个值大/小
SELECT * FROM EMP WHERE sal > ANY (SELECT sal FROM EMP WHERE deptno = 30);
```

**多列子查询**

```sql
SELECT ename FROM EMP
WHERE (deptno, job) = (SELECT deptno, job FROM EMP WHERE ename = 'SMITH')
    AND ename <> 'SMITH';
```

**在 FROM 中使用子查询**

```sql
SELECT DEPT.deptno, DEPT.dname, tmp.emp_count
FROM DEPT,
    (SELECT COUNT(*) AS emp_count, deptno
     FROM EMP GROUP BY deptno) tmp
WHERE DEPT.deptno = tmp.deptno;
```

### 5. 合并查询

| 操作符 | 说明 |
|--------|------|
| `UNION` | 合并去重 |
| `UNION ALL` | 合并不去重 |

```sql
SELECT ename, sal, job FROM EMP WHERE sal > 2500
UNION
SELECT ename, sal, job FROM EMP WHERE job = 'MANAGER';
```

### 6. 内连接与外连接

**内连接**：只保留匹配记录

**左外连接**：完全显示左表，右表无匹配为 NULL

```sql
SELECT stu.id, stu.name, exam.score
FROM stu
LEFT JOIN exam ON stu.id = exam.stu_id;
```

**右外连接**：同理，完全显示右表

### 7. 全外连接 (FULL OUTER JOIN)

全外连接返回两个表的所有记录，未匹配的左表记录补 NULL，未匹配的右表记录也补 NULL。

```sql
SELECT stu.id, stu.name, exam.score
FROM stu
FULL OUTER JOIN exam ON stu.id = exam.stu_id;
```

**MySQL 不直接支持 FULL OUTER JOIN**，可通过 UNION 模拟：

```sql
SELECT stu.id, stu.name, exam.score
FROM stu
LEFT JOIN exam ON stu.id = exam.stu_id
UNION
SELECT stu.id, stu.name, exam.score
FROM stu
RIGHT JOIN exam ON stu.id = exam.stu_id;
```

**四种外连接对比**：

| 连接类型 | 左表全部 | 右表全部 | 匹配部分 |
|----------|----------|----------|----------|
| INNER JOIN | ❌ | ❌ | ✅ |
| LEFT JOIN | ✅ | ❌ | ✅ |
| RIGHT JOIN | ❌ | ✅ | ✅ |
| FULL OUTER JOIN | ✅ | ✅ | ✅ |

---

## 八、 视图

视图是从一个或多个表导出的虚拟表，其内容由查询定义。

### 1. 视图的创建与使用

```sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

### 2. 视图的特点

* **虚拟性**：视图不存储实际数据，数据存放在基表中
* **实时性**：视图会实时反映基表数据的变化
* **简化性**：封装复杂查询，供多次使用
* **安全性**：可限制用户访问敏感字段

### 3. 视图分类

**可更新视图**：基于单表、无聚合/分组/ DISTINCT/ UNION 的视图可更新

```sql
CREATE VIEW view_student AS
SELECT id, name, class_id FROM student WHERE class_id = 1;

INSERT INTO view_student (id, name, class_id) VALUES (100, '张三', 1);
```

**只读视图**：使用 `WITH CHECK OPTION` 限制 DML 操作范围

```sql
CREATE VIEW view_student AS
SELECT * FROM student WHERE class_id = 1
WITH CHECK OPTION;
```

### 4. 删除视图

```sql
DROP VIEW IF EXISTS view_name;
```

### 5. 视图使用场景

| 场景 | 说明 |
|------|------|
| 简化复杂查询 | 将多表连接封装为视图 |
| 列级权限控制 | 隐藏敏感字段供特定用户访问 |
| 业务逻辑抽象 | 将经常变化的业务规则封装在视图层 |

---

## 九、 事务

事务是数据库操作的最小工作单元，事务内的所有操作要么全部成功，要么全部失败。

### 1. 事务四大特性 (ACID)

| 特性 | 说明 | 实现原理 |
|------|------|----------|
| **原子性 (Atomicity)** | 事务是不可分割的最小单元 | Undo Log 回滚 |
| **一致性 (Consistency)** | 事务执行前后数据库状态保持一致 | 约束规则保证 |
| **隔离性 (Isolation)** | 并发事务相互隔离，不互相干扰 | 锁机制 + MVCC |
| **持久性 (Durability)** | 事务提交后数据永久保存 | Redo Log 写入 |

### 2. 事务控制语句

```sql
-- 开启事务
START TRANSACTION;  -- 或 BEGIN

-- 提交事务
COMMIT;

-- 回滚事务
ROLLBACK;

-- 设置保存点
SAVEPOINT point1;

-- 回滚到保存点
ROLLBACK TO SAVEPOINT point1;

-- 释放保存点
RELEASE SAVEPOINT point1;
```

### 3. 事务隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|----------|------|------------|------|
| **READ UNCOMMITTED** | ✅ 可能 | ✅ 可能 | ✅ 可能 |
| **READ COMMITTED** | ❌ 不可能 | ✅ 可能 | ✅ 可能 |
| **REPEATABLE READ** (MySQL默认) | ❌ 不可能 | ❌ 不可能 | ✅ 可能 |
| **SERIALIZABLE** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能 |

**设置隔离级别**：

```sql
-- 会话级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 全局级别
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 4. 事务使用场景

**转账场景**：

```sql
START TRANSACTION;

UPDATE account SET balance = balance - 1000 WHERE user_id = 1;
UPDATE account SET balance = balance + 1000 WHERE user_id = 2;

-- 检查业务逻辑
IF @@ERROR = 0 THEN
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

**订单创建场景**：

```sql
START TRANSACTION;

INSERT INTO orders (user_id, total) VALUES (1, 500);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 101, 2);
UPDATE stock SET quantity = quantity - 2 WHERE product_id = 101;

COMMIT;
```

### 5. 事务并发问题

* **脏读**：读取到其他事务未提交的数据
* **不可重复读**：同一事务内两次读取同一行数据结果不同（其他事务修改了数据）
* **幻读**：同一事务内两次查询结果集不同（其他事务插入/删除了行）

### 6. 事务注意事项

* 事务范围不宜过大，否则占用锁资源导致并发下降
* 长时间事务可能导致主从延迟
* InnoDB 支持行级锁，MyISAM 不支持事务

---

## 十、 连接池

### 1. 短连接的问题

传统短连接每次操作都新建连接，资源消耗大。

| 问题 | 说明 |
|------|------|
| 连接耗时 | TCP 三次握手 + MySQL 认证，耗时 10-100ms |
| 资源消耗 | 每个连接占用约 1MB 内存 |
| 服务器压力 | MySQL 单机通常支持 150-200 并发连接 |

### 2. 连接池原理

* **提前初始化**：启动时创建一组数据库连接
* **资源池化**：复用连接，避免频繁创建/销毁
* **按需取用**：用完归还供其他请求使用

### 3. 主流连接池对比

| 连接池 | 特性 | 适用场景 |
|--------|------|----------|
| **Druid** (阿里巴巴) | 监控能力强，支持 SQL 防火墙 | 企业级应用 |
| **HikariCP** | 性能最优，轻量级 | 高并发场景 |
| **C3P0** | 老牌连接池，功能全面 | 遗留系统 |

### 4. 连接池核心参数

```properties
# 最小空闲连接数
minimum-idle=5
# 最大连接数
maximum-pool-size=20
# 连接超时时间(ms)
connection-timeout=30000
# 空闲存活时间(ms)
idle-timeout=600000
# 连接最大生命周期(ms)
max-lifetime=1800000
```

### 5. 连接池工作流程

```
请求进入 → 检查池中是否有空闲连接
    ├── 有 → 取出使用 → 使用完毕归还池中
    └── 无 → 检查是否已达最大连接数
        ├── 未达 → 创建新连接 → 使用 → 归还
        └── 已达 → 等待/超时
```

---

## 十一、 SQL 执行顺序

```sql
FROM > ON > JOIN > WHERE > GROUP BY > HAVING > SELECT > DISTINCT > ORDER BY > LIMIT
```

---

## 十二、 InnoDB 与 MyISAM 对比

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 | ❌ 不支持 |
| 锁粒度 | 行级锁 | 表级锁 |
| 外键支持 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | 自动恢复 | 需手动 |
| COUNT(*) | 全表扫描 | 内存计数器 |

---

## 十三、 注意事项

1. **NULL 运算**：任何数据与 NULL 运算结果为 NULL
2. **HAVING 使用**：聚合函数不能在 WHERE 中使用，需用 HAVING
3. **事务回滚**：TRUNCATE 不记录日志，无法回滚