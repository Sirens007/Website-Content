---
title: MySQL数据库：事务
published: 2026-08-07
pinned: false
description: "系统学习MySQL事务、ACID、隔离级别、MVCC机制."
image: ./mysql.png
tags: ["MySQL"]
category: MySQL
draft: false
---


> 适用环境：MySQL 8.0，主要讨论 InnoDB
>
> 事务不是把 SQL 写在一起就结束，而是为一组相关操作划定“共同成功或共同失败”的边界。

---

## 1. 事务

事务（Transaction）是数据库执行工作的逻辑单位，可以包含一条或多条 SQL。事务结束只有两种主要结果：

- `COMMIT`：确认全部修改，使结果持久化；
- `ROLLBACK`：撤销本事务尚未提交的修改；

例如“张三向李四转账 100 元”至少包含扣款和入账两次更新。如果扣款成功后程序异常，而入账没有执行，总金额就会错误；所以两次更新必须属于同一事务

事务能力由存储引擎提供；InnoDB 支持事务，MyISAM 不支持，先检查表的引擎：

```sql
show engines;
show table status like 'bank_account';
show create table bank_account;
```

一次事务最好只操作支持事务的表。若混入非事务表，`ROLLBACK` 不能撤销其修改，会破坏“共同成功或失败”。

---

## 2. ACID 特性

| 特性 | 含义 | 转账中的表现 | InnoDB 的主要实现 |
| --- | --- | --- | --- |
| Atomicity 原子性 | 操作不可只完成一部分 | 扣款和入账一起成功或撤销 | undo log、回滚机制 |
| Consistency 一致性 | 事务前后都满足数据与业务规则 | 总额不变、余额合法 | 约束、应用规则，以及其他三项共同保证 |
| Isolation 隔离性 | 并发事务尽量像互不干扰地执行 | 转账中间状态不被错误使用 | MVCC、锁、隔离级别 |
| Durability 持久性 | 成功提交的结果能在故障后恢复 | 返回成功后转账不会因宕机消失 | redo log、刷盘与崩溃恢复 |

一致性不是“开启事务后数据库自动理解全部业务”。数据库可保证主键、外键、`CHECK` 等规则，但“转账总额不变”“库存不能超卖”等业务不变量仍要由正确 SQL、锁定方式和应用判断共同保证。

---

## 3. 基本语法与事务边界

```sql
show engines; 	    -- 查看支持事务的存储引擎

start transaction;  -- 从这里开始，后面的修改先不要自动提交
-- 或 begin;

-- 执行 select / insert / update / delete 后
commit;             -- 提交并结束事务/永久保存

rollback;			-- 回滚当前事务/撤销修改
```

`START TRANSACTION` 与 `BEGIN` 在普通 SQL 中都能开启事务；前者还能使用：

```sql
start transaction read only;
start transaction read write;
start transaction with consistent snapshot;
```

`READ ONLY` 适合明确的只读事务，InnoDB 可据此优化。`WITH CONSISTENT SNAPSHOT` 在 `REPEATABLE READ` 下立即建立一致性快照。

MySQL 不支持普通事务嵌套。在未结束的事务中再次执行 `START TRANSACTION`，会先隐式提交原事务，不是创建子事务。存储过程里的 `BEGIN ... END` 表示程序块；需要开启事务时应明确写 `START TRANSACTION`。

---

## 4. 转账事务实操

### 4.1 开启事务，执行修改后回滚

开启事务

```sql
# 开启事务
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

# 在修改之前查看表中的数据
mysql> SELECT * FROM bank_account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | 张三 | 1000.00 |
|  2 | 李四 | 1000.00 |
+----+------+---------+
2 rows in set (0.00 sec)
```

执行修改

```sql
# 张三余额减少 100
mysql> UPDATE bank_account
       SET balance = balance - 100
       WHERE name = '张三';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

# 李四余额增加 100
mysql> UPDATE bank_account
       SET balance = balance + 100
       WHERE name = '李四';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

# 修改之后、回滚之前查看表中的数据
mysql> SELECT * FROM bank_account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | 张三 |  900.00 |
|  2 | 李四 | 1100.00 |
+----+------+---------+
2 rows in set (0.00 sec)
```

回滚事务

```sql
# 回滚事务
mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)

# 再次查询，发现修改没有生效
mysql> SELECT * FROM bank_account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | 张三 | 1000.00 |
|  2 | 李四 | 1000.00 |
+----+------+---------+
2 rows in set (0.00 sec)
```

### 4.2 开启事务，执行修改后提交

开启事务

```sql
# 开启事务
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

# 在修改之前查看表中的数据
mysql> SELECT * FROM bank_account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | 张三 | 1000.00 |
|  2 | 李四 | 1000.00 |
+----+------+---------+
2 rows in set (0.00 sec)

```

执行修改

```sql
# 张三余额减少 100
mysql> UPDATE bank_account
       SET balance = balance - 100
       WHERE name = '张三';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

# 李四余额增加 100
mysql> UPDATE bank_account
       SET balance = balance + 100
       WHERE name = '李四';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

# 修改之后、提交之前查看表中的数据
mysql> SELECT * FROM bank_account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | 张三 |  900.00 |
|  2 | 李四 | 1100.00 |
+----+------+---------+
2 rows in set (0.00 sec)
```

提交事务

```sql
# 提交事务
mysql> COMMIT;
Query OK, 0 rows affected (0.01 sec)

# 再次查询，数据已经被修改，说明数据已经持久化到磁盘
mysql> SELECT * FROM bank_account;
+----+------+---------+
| id | name | balance |
+----+------+---------+
|  1 | 张三 |  900.00 |
|  2 | 李四 | 1100.00 |
+----+------+---------+
2 rows in set (0.00 sec)
```

---

## 5. 自动提交与隐式提交

MySQL 新连接默认开启自动提交。未显式开启事务时，每条成功语句各自形成一个事务，执行后立即提交：

```sql
select @@session.autocommit;	-- 查看当前设置
set autocommit = 0;  -- 当前会话关闭自动提交
set autocommit = 1;  -- 恢复自动提交
```

两种常用方式：

```sql
-- 方式一：保留默认自动提交，只临时开启一次事务
start transaction;
update bank_account set balance = balance + 10 where id = 1;
commit;

-- 方式二：持续关闭当前连接的自动提交
set autocommit = 0;
update bank_account set balance = balance - 10 where id = 1;
commit;              -- 之后仍需继续显式提交或回滚
```

当 `autocommit=0` 时，会话结束而最后一个事务未提交，MySQL 会回滚它；从 `0` 改成 `1` 会隐式提交当前事务。

以下语句通常会隐式提交当前事务，不能指望随后 `ROLLBACK` 撤销：

- DDL：`CREATE`、`ALTER`、`DROP`、`TRUNCATE`、`RENAME`；
- 部分账户、管理和锁表语句；
- 再次执行 `START TRANSACTION`，或执行使值从 0 变为 1 的 `SET autocommit=1`。

注意：

> [!CAUTION]
>
> - 只要使用 `start transaction` 或 `begin` 开启事务，必须要通过 `commit` 提交才会持久化，与是否设置 `set autocommit` 无关；
> - 手动提交模式下，不用显式开启事务，执行修改操作后，提交或回滚事务时直接使用`commit` 或 `rollback`；
> - 已提交的事务不能回滚；

---

## 6. 保存点

涉及语法：

```sql
savepoint 保存点名称;
```

回滚到指定保存点：

```sql
rollback to 保存点名称;
```

例如：

保存点用于只撤销事务后半段，不等于提交前半段：

```sql
start transaction;
savepoint before_transfer;

update bank_account set balance = balance - 100 where id = 1;
update bank_account set balance = balance + 100 where id = 2;
rollback to savepoint before_transfer;  -- 撤销上面两次更新

-- 改用 50 元重新执行
update bank_account set balance = balance - 50 where id = 1;
update bank_account set balance = balance + 50 where id = 2;

release savepoint before_transfer;      -- 删除保存点，不提交事务
commit;
```

`ROLLBACK TO SAVEPOINT` 不结束事务；目标保存点之后创建的保存点会被删除。`COMMIT` 或不带保存点的 `ROLLBACK` **才会结束事务并清除全部保存点**。InnoDB 回滚到保存点时不保证释放之后取得的全部内存行锁，因此保存点不能代替“缩短事务”。

---

## 7. 事务的隔离性和四种隔离级别

### 7.1 隔离性

MySQL服务被多个客户端同时访问，每个客户端执行的DML语句以事务为基本单位，那么不同的客户端在对同一张表中的**同一条数据进行修改时就可能出现相互影响**的情况，为保证不同事务直接执行时不受影响，因此事务之间就需要相互隔离

### 7.2 隔离级别及设置语法

| 隔离级别 | 脏读 | 不可重复读 | 幻读（SQL 标准） | InnoDB 主要行为 |
| --- | --- | --- | --- | --- |
| `READ UNCOMMITTED` 读未提交 | 可能 | 可能 | 可能 | 普通读可能看到未提交版本，业务中很少使用 |
| `READ COMMITTED` 读已提交 | 避免 | 可能 | 可能 | 每条一致性读建立新快照；通常不使用间隙锁 |
| `REPEATABLE READ` 可重复读 | 避免 | 避免 | 标准允许 | InnoDB 默认；普通读复用快照，范围锁定读用 Next-Key Lock 防典型幻读 |
| `SERIALIZABLE` 串行化 | 避免 | 避免 | 避免 | 显式事务中的普通读会按共享锁读处理，冲突等待更多 |

从上往下：并发性能越低，隔离度（安全性）越高



隔离越强不代表所有事务真的在全库中逐个执行。即使是 `SERIALIZABLE`，互不冲突的事务仍可并发；它主要通过更严格的锁让冲突操作等待，因此吞吐量通常更低。

涉及语法：

1. 查看全局隔离级别

   ```sql
   select @@global.transaction_isolation;
   ```

2. 查看当前会话隔离级别

   ```sql
   select @@session.transaction_isolation;
   ```

**以下是设置SESSION隔离级别**

设置事务的隔离级别和访问模式，可以使用以下语法：

```sql
# 通过 GLOBAL | SESSION 分别指定不同作用域
SET [GLOBAL | SESSION] TRANSACTION transaction_characteristic;

transaction_characteristic: {
      ISOLATION LEVEL level
    | access_mode
}

# 隔离级别
level: {
      REPEATABLE READ     # 可重复读
    | READ COMMITTED      # 读已提交
    | READ UNCOMMITTED    # 读未提交
    | SERIALIZABLE        # 串行化
}

# 访问模式
access_mode: {
      READ WRITE          # 事务可以读取和修改数据
    | READ ONLY           # 事务只能读取数据，不能修改普通表中的数据
}

# 示例

# 设置全局事务隔离级别为串行化
# 对之后新建的连接生效，不影响当前及已有连接
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;

# 设置当前会话的事务隔离级别为串行化
# 对当前会话后续的所有事务生效，不影响正在执行的事务
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

# 不指定作用域时，只对当前会话的下一个事务生效
# 随后的事务恢复为之前的会话隔离级别
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

```sql
# 方式一：直接设置系统变量
SET GLOBAL transaction_isolation = 'SERIALIZABLE';

# 隔离级别中原本使用空格的位置，需要改成连字符
SET SESSION transaction_isolation = 'REPEATABLE-READ';


# 方式二：使用 @@GLOBAL 和 @@SESSION 设置系统变量
SET @@GLOBAL.transaction_isolation = 'SERIALIZABLE';

# 隔离级别中原本使用空格的位置，需要改成连字符
SET @@SESSION.transaction_isolation = 'REPEATABLE-READ';
```

使用系统变量赋值时值中写连字符，如 `SET SESSION transaction_isolation='READ-COMMITTED'`；使用 `SET TRANSACTION ... LEVEL` 语法时写空格。改变隔离级别前应确认作用域，做双会话实验时直接设置 `SESSION` 最清楚。

### 7.3 脏读示例

> **脏读**：一个事务读取到了另一个事务尚未提交的数据。

客户端 A 设置 READ UNCOMMITTED

客户端 B 设置 READ UNCOMMITTED

**客户端 A 开启事务并修改，但不提交**

```sql
start transaction;

update bank_account
set balance = 100000
where name = '王五';

select *
from bank_account
where name = '王五';
```

客户端 A 自己看到：王五 100000.00

**客户端 B 读取数据**

```sql
start transaction;

select *
from bank_account
where name = '王五';
```

在 `READ UNCOMMITTED` 下，按照资料的实验逻辑，客户端 B 可以看到事务A尚未提交的修改：王五 100000.00

此时发生了**脏读**

```txt
客户端A                         客户端B
事务A                           事务B

START TRANSACTION
     │
UPDATE 王五=100000
     │
没有COMMIT
     │
     ├──────────────────────→ SELECT
     │                         读到100000
     │
ROLLBACK
     │
王五恢复2000
                               SELECT
                               读到2000
```

### 7.4 不可重复读示例

> **不可重复读**：同一个事务中，使用相同条件读取同一行数据，两次得到的字段值不同。

```txt
客户端A                             客户端B
事务A                               事务B

START TRANSACTION
    │
SELECT 王五
    │
    └── 2000

                                START TRANSACTION
                                       │
                                UPDATE 王五=1000
                                       │
                                   尚未COMMIT

SELECT 王五
    │
    └── 2000
        ↑
        看不到B未提交的数据

                                   COMMIT
                                     │
                                     ↓
                                1000正式提交

SELECT 王五
    │
    └── 1000

事务A中：
第一次 = 2000
第二次 = 1000

→ 不可重复读
事务 A 中的两条查询完全相同，但第一次得到 2000，第二次得到 1000，因此称为“不可重复读”。
```

因为 InnoDB 在 `READ COMMITTED` 下，每次普通一致性查询都会建立一个新的数据快照，所以第二次查询可以看到其他事务新提交的修改。

在 `REPEATABLE READ` 下，同一个事务中的普通 `SELECT` 通常使用第一次查询建立的快照，所以能够保持重复读取结果一致。

### 7.5 幻读示例

> **幻读**：同一个事务中，使用相同范围条件查询两次，第二次查询出现或消失了一些**符合条件的记录**

假设第一次查询余额不低于 1000 元的账户：

```
-- 事务 A
start transaction;

select *
from bank_account
where balance >= 1000;
```

第一次结果有两行：

| id   | name | balance |
| ---- | ---- | ------- |
| 1    | 张三 | 1000    |
| 2    | 李四 | 1000    |

然后事务 B 插入一条符合条件的数据并提交：

```
-- 事务 B
start transaction;

insert into bank_account (name, balance)
values ('王五', 2000);

commit;
```

事务 A 再次执行相同查询：

```
select *
from bank_account
where balance >= 1000;
```

如果第二次结果变成三行：

| id   | name | balance |
| ---- | ---- | ------- |
| 1    | 张三 | 1000    |
| 2    | 李四 | 1000    |
| 3    | 王五 | 2000    |

事务 A 会感觉像凭空出现了一条记录，因此称为“幻读”。

**注意：**

不可重复读是**指同一行的值变**了，而幻读是**结果集中的行变**了。

**不可重复读**通常是由另一个事务执行 `update` 并提交引起；

**幻读**通常是由另一个事务执行 `insert` 或 `delete` 并提交引起。



**但 MySQL InnoDB 比较特殊**

在InnoDB的 REPEATABLE READ 下，典型的幻读通常已经被解决。涉及到MVCC，Next-Key Lock。

### 7.6 串行化 SERIALIZABLE 解释

尽量让事务表现得像**一个一个顺序执行**。

```txt
事务A
  ↓
执行完成

事务B
  ↓
执行完成

事务C
  ↓
执行完成
```

---

## 8. MVCC 与一致性读原理

MVCC（多版本并发控制）让读操作在许多情况下不必阻塞写操作。InnoDB 聚集索引记录内部包含：

- `DB_TRX_ID`：最后一次插入或更新该行的事务 ID；
- `DB_ROLL_PTR`：指向 undo log 中的旧版本；
- 没有合适聚集键时还会使用隐藏的 `DB_ROW_ID`。

```mermaid
flowchart LR
    Q["一致性读与 Read View"] --> C["当前版本：trx 120"]
    C --> U1["undo 旧版本：trx 105"]
    U1 --> U2["undo 更旧版本：trx 88"]
    Q -.-> U1
```

查询读取当前记录后，用 Read View 判断创建该版本的事务对自己是否可见；不可见就沿 `DB_ROLL_PTR` 在 undo 版本链中寻找第一个可见版本。Read View 可近似理解为“生成快照时哪些事务已提交、哪些仍活跃”的可见性规则，不是复制出一整份数据库。

- `READ COMMITTED`：每条普通 `SELECT` 建立新 Read View，所以同一事务后一次查询可能看到新提交；
- `REPEATABLE READ`：通常由第一次一致性读建立 Read View，事务内后续普通读复用它；
- 当前事务自己的修改始终可见。

长事务会让旧版本长期不能清理，使 undo 膨胀并增加版本链读取成本，所以不要无故让事务长时间保持开启。

---

## 参考

- [MySQL 8.0：事务的开始、提交与回滚](https://dev.mysql.com/doc/refman/8.0/en/commit.html)
- [MySQL 8.0：事务隔离级别](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [MySQL 8.0：一致性非锁定读](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)
- [MySQL 8.0：InnoDB 锁](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL 8.0：MVCC](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [MySQL 8.0：隐式提交](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)
