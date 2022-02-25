# 隔离级别

## Serializable 可序列化
## Repeatable Read 可重复度（默认）
每次select从事务开始时的快照读取
## Read Committed 读已提交
每次select从当前快照读取
Gap锁会被禁用，会出幻读
## Read Uncommitted 读未提交

## 概念

## 幻读
在一个事务中，同一个查询在不同时间产生不同的行集。

### 问题：
1.跟不可重复读有什么区别？

2.通过锁解决还是MVCC？

# Multi-version concurrency control (MVCC) 
## desc
信息存储在undo log，用于回滚、构建数据的早期版本。

每行有三列：

- DB_TRX_ID：标识上一次对本行进行inserted或updated（deletion）的事务id。
- DB_ROLL_PTR：回滚指针指向写入回滚段的撤消日志记录
- DB_ROW_ID： 一个字增的行id

### 问题：
1.索引会有版本控制么？

# 索引
## desc
聚簇索引or二级索引

聚簇索引使用主键、第一个唯一索引+所有非空列、隐藏的GEN_CLUST_INDEX

二级索引保存该索引列的值和主键，用于在聚簇索引内查询（回表）

### 问题：
1.索引是如何更新的？

二级索引：Change Buffer,to be continue

# 锁

## row-level (index)
共享锁/排他锁

Record Locks

Gap Locks

Next-Key Locks

在使用唯一索引的等于条件时不使用，在Read Committed中不使用。
在查询包含多列唯一索引中部分列使用，在非唯一索引或无索引的的等于条件使用。

## 实验记录

实验：空表test_mysql，a列主键，b列有非唯一索引，c列无索引。

现有数据：

```
+---+-----+---+
| a | b   | c |
+---+-----+---+
| 0 | 100 | 1 |
| 1 | 101 | 2 |
| 2 | 102 | 1 |
| 3 | 103 | 1 |
| 4 | 104 | 1 |
| 5 | 105 | 1 |
| 6 | 106 | 3 |
+---+-----+---+
```

### REPEATABLE READ（开启Gap Lock）：

#### 对于主键列

##### 等值查询

事务1：```select * from x where a = 7 for update```，没有匹配行。
```
mysql> select ENGINE_TRANSACTION_ID as trx_id, OBJECT_SCHEMA, OBJECT_NAME, INDEX_NAME, OBJECT_INSTANCE_BEGIN, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA from performance_schema.data_locks;
+--------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| trx_id | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA |
+--------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+-----------+
|   4438 | test_mysql    | test_lock   | NULL       |       140341479068080 | TABLE     | IX        | GRANTED     | NULL      |
|   4438 | test_mysql    | test_lock   | PRIMARY    |       140341479065168 | RECORD    | X         | GRANTED     | 2         |
|   4438 | test_mysql    | test_lock   | PRIMARY    |       140341479065168 | RECORD    | X         | GRANTED     | 5         |
|   4438 | test_mysql    | test_lock   | PRIMARY    |       140341479065512 | RECORD    | X,GAP     | GRANTED     | 8         |
+--------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+-----------+
4 rows in set (0.02 sec)
```

事务2：```insert into x values (1, 101, 1)```，等待事务1提交。

#### 对于非唯一索引列

事务1：```select * from x where b = 101 for update```，没有匹配行，除会添加伪记录（supremum pseudo-record）上X的Record Lock外，还会X锁全部匹配列。

事务2：```insert into x values (1, 101, 1)```，等待事务1提交。

#### 对于无索引列

事务1：```select * from x where c = 1 for update```，没有匹配行，除会添加伪记录（supremum pseudo-record）上X的Record Lock外，还会X锁全表。

事务2：```insert into x values (1, 101, 1)```，等待事务1提交。

READ COMMITTED（关闭Gap Lock）：

对于a列：

事务1：`select * from x where a = 7 for update`，没有匹配行，没有上Record Lock，没有上Gap Lock。

事务2：`insert into x values (7, 107, 7)`，直接成功。

### 问题：
1.锁行还是锁索引？

锁索引。

2.为什么在同一段gap上允许两个事务持有冲突的Gap Lock?

Gap Lock的唯一目的是防止其他事务插入间隙。


## table-level
意向锁 IS IX

在一个事务可以获得一行的`共享锁`前，必须获得该表的`IS锁`或`更强的锁`

在一个事务可以获得一行的`排他锁`前，必须获得该表的`IX锁`



# 注
- select * from performance_schema.data_locks;
- select ENGINE_TRANSACTION_ID as trx_id, OBJECT_SCHEMA, OBJECT_NAME, INDEX_NAME, OBJECT_INSTANCE_BEGIN, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA from performance_schema.data_locks;
- SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
- show variables like '%general_log%';
- show variables like '%log_output%'; 
> SET GLOBAL log_output = 'FILE';  SET GLOBAL general_log = 'ON';   //日志开启（日志输出到文件）
> SET GLOBAL log_output = 'FILE';  SET GLOBAL general_log = 'OFF';  //日志关闭
或者
> SET GLOBAL log_output = 'TABLE'; SET GLOBAL general_log = 'ON';   //日志开启（日志输出到表：mysql.general_log）
> SET GLOBAL log_output = 'TABLE'; SET GLOBAL general_log = 'OFF';  //日志关闭