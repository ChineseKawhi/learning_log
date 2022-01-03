# 分类
## row-level (index)
- 共享锁/排他锁

- Record Locks

- Gap Locks

- Next-Key Locks (Gap Locks + Record Locks)

- Insert Intention Locks (Gap Locks)

在使用唯一索引的等于条件时不使用，在Read Committed中不使用。
在查询包含多列唯一索引中部分列使用，在非唯一索引或无索引的的等于条件使用。

## table-level
- 意向锁 IS IX

- AUTO-INC Locks

在一个事务可以获得一行的`共享锁`前，必须获得该表的`IS锁`或`更强的锁`

在一个事务可以获得一行的`排他锁`前，必须获得该表的`IX锁`

# 上锁流程

按照索引查找，先上Next-Key Locks，不匹配逐步取消Gap Locks与Record Locks。

# 实验记录

实验：

表结构：
```
mysql> desc test_lock;
+-------+------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+-------+------+------+-----+---------+-------+
| a     | int  | NO   | PRI | NULL    |       |
| b     | int  | NO   | UNI | NULL    |       |
| c     | int  | NO   |     | NULL    |       |
| d     | int  | NO   | MUL | NULL    |       |
+-------+------+------+-----+---------+-------+
4 rows in set (0.01 sec)
```

现有数据：

```
mysql> select * from test_lock;
+---+----+-----+---+
| a | b  | c   | d |
+---+----+-----+---+
| 2 | 12 | 102 | 2 |
| 5 | 15 | 105 | 5 |
| 8 | 18 | 108 | 8 |
+---+----+-----+---+
3 rows in set (0.01 sec)
```

## REPEATABLE READ（开启Gap Lock）：

### 对于主键列
1. 等值查询
- 无匹配行:

    事务1：

    ```select * from test_lock where a = 7 for update```
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   4515 | NULL       | TABLE     | IX        | NULL      |
    |   4515 | PRIMARY    | RECORD    | X,GAP     | 8         |
    +--------+------------+-----------+-----------+-----------+
    2 rows in set (0.01 sec)
    ```

    事务2：

    ```insert into test_lock values(7, 17,107,7)```，等待事务1提交。

    ```insert into test_lock values(6, 16,106,6)```，等待事务1提交。

    ```update test_lock set d = 99 where a = 5```，直接成功。

    结论：等值查询无匹配行，上目前匹配查找到的范围的GAP锁，即a=5至a=8（不包括边界行），插入在范围内但a不等于7的行，同样要等待。

- 有匹配行
    ```select * from test_lock where a = 5 for update```
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4615 | NULL       | TABLE     | IX            | NULL      |
    |   4615 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         |
    +--------+------------+-----------+---------------+-----------+
    2 rows in set (0.00 sec)
    ```

    事务2：

    ```insert into test_lock values(3, 13,103,3)```，直接成功。

    ```insert into test_lock values(6, 16,106,6)```，直接成功。


    结论：等值查有匹配行，上目前匹配行的Record Lock，即a=5。

- 更改

    ```update test_lock set a=7 where a=5;```
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4719 | NULL       | TABLE     | IX            | NULL      |
    |   4719 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         |
    |   4719 | b_index    | RECORD    | X,REC_NOT_GAP | 15, 5     |
    |   4719 | b_index    | RECORD    | S,GAP         | 15, 5     |
    |   4719 | b_index    | RECORD    | S,GAP         | 18, 8     |
    |   4719 | b_index    | RECORD    | S,GAP         | 15, 7     |
    +--------+------------+-----------+---------------+-----------+
    6 rows in set (0.02 sec)
    ```

    事务2：

    ```select * from test_lock where b < 13 for share;```，等待事务1。

    ```select * from test_lock where b < 12 for share;```，直接成功。

    事务1:

    ```update test_lock set a=7 where b=15;```
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4713 | NULL       | TABLE     | IX            | NULL      |
    |   4713 | b_index    | RECORD    | X,REC_NOT_GAP | 15, 5     |
    |   4713 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         |
    |   4713 | b_index    | RECORD    | S,GAP         | 15, 5     |
    |   4713 | b_index    | RECORD    | S,GAP         | 18, 8     |
    |   4713 | b_index    | RECORD    | S,GAP         | 15, 7     |
    +--------+------------+-----------+---------------+-----------+
    6 rows in set (0.01 sec)
    ```

    事务2：

    ```select * from test_lock where b < 13 for share;```，等待事务1。

    ```select * from test_lock where b < 12 for share;```，直接成功。

    问题：为什么```select * from test_lock where b < 13 for share;```需要等待？

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA, LOCK_STATUS from performance_schema.data_locks;
    +-----------------+------------+-----------+---------------+-----------+-------------+
    | trx_id          | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA | LOCK_STATUS |
    +-----------------+------------+-----------+---------------+-----------+-------------+
    |            4723 | NULL       | TABLE     | IX            | NULL      | GRANTED     |
    |            4723 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         | GRANTED     |
    |            4723 | b_index    | RECORD    | X,REC_NOT_GAP | 15, 5     | GRANTED     |
    |            4723 | b_index    | RECORD    | S,GAP         | 15, 5     | GRANTED     |
    |            4723 | b_index    | RECORD    | S,GAP         | 18, 8     | GRANTED     |
    |            4723 | b_index    | RECORD    | S,GAP         | 15, 7     | GRANTED     |
    | 421816565330728 | NULL       | TABLE     | IS            | NULL      | GRANTED     |
    | 421816565330728 | b_index    | RECORD    | S             | 12, 2     | GRANTED     |
    | 421816565330728 | PRIMARY    | RECORD    | S,REC_NOT_GAP | 2         | GRANTED     |
    | 421816565330728 | b_index    | RECORD    | S             | 15, 5     | WAITING     |
    +-----------------+------------+-----------+---------------+-----------+-------------+
    10 rows in set (0.01 sec)
    ```
    发现最后一行事务2在请求b=15的Next-Key Lock，由此可知在扫描时使用Next-Key Lock逐步加锁，参考二级索引等值查询有匹配行，在Next-Key Lock加锁成功后，再根据条件释放Gap Lock与Record Lock，但b=15行此时有Record Lock，只能Waiting。

    结论：对主键进行更改时，会S锁定二级唯一索引的相关GAP(b=12的两端GAP)，二级唯一索引中原行锁包括两端GAP，原行Record，目标行包括左侧GAP。此时GAP内无法插入，只能读取。

2. 非等值查询
- 无匹配行

    事务1：

    ```select * from test_lock where a > 99 for update```

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+------------------------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA              |
    +--------+------------+-----------+-----------+------------------------+
    |   4636 | NULL       | TABLE     | IX        | NULL                   |
    |   4636 | PRIMARY    | RECORD    | X         | supremum pseudo-record |
    +--------+------------+-----------+-----------+------------------------+
    2 rows in set (0.00 sec)
    ```

    事务2：

    ```insert into test_lock values (11,21,121,21);```，等待事务1提交。

    ``` update test_lock set d = 99 where a = 8;```，直接成功。

    结论：非等值查询，锁了当前最大主键到$\infty$的Next-Key Lock，不包括当前最大主键。
 
    -----

    事务1：

    ```select * from test_lock where a < 2 for update```

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   5143 | NULL       | TABLE     | IX        | NULL      |
    |   5143 | PRIMARY    | RECORD    | X,GAP     | 2         |
    +--------+------------+-----------+-----------+-----------+
    2 rows in set (0.01 sec)
    ```

    事务2：

    ```select * from test_lock where a < 2 for update;```，执行成功。
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   5144 | NULL       | TABLE     | IX        | NULL      |
    |   5144 | PRIMARY    | RECORD    | X,GAP     | 2         |
    |   5143 | NULL       | TABLE     | IX        | NULL      |
    |   5143 | PRIMARY    | RECORD    | X,GAP     | 2         |
    +--------+------------+-----------+-----------+-----------+
    4 rows in set (0.00 sec)
    ```
    此时，若事物1/2都执行```insert into test_lock values (1,11,101,1);```，先执行的事务阻塞，后执行的事务报错```ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction```重启，此时锁如下：
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+------------------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE              | LOCK_DATA |
    +--------+------------+-----------+------------------------+-----------+
    |   5143 | NULL       | TABLE     | IX                     | NULL      |
    |   5143 | PRIMARY    | RECORD    | X,GAP                  | 2         |
    |   5143 | PRIMARY    | RECORD    | X,GAP                  | 1         |
    |   5143 | PRIMARY    | RECORD    | X,GAP,INSERT_INTENTION | 2         |
    +--------+------------+-----------+------------------------+-----------+
    4 rows in set (0.00 sec)
    ```

    结论：X,GAP锁之间并不会互相排斥，根据文档GAP锁仅用于防止插入，即只和X,INSERT_INTENTION锁互斥，事务1，2都能拿到X,GAP，但在进行插入时，都无法获得X,INSERT_INTENTION锁，形成死锁。


- 有匹配行

    事务1：

    ```select * from test_lock where a < 7 and a > 3 for update```

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   4512 | NULL       | TABLE     | IX        | NULL      |
    |   4512 | PRIMARY    | RECORD    | X         | 5         |
    |   4512 | PRIMARY    | RECORD    | X,GAP     | 8         |
    +--------+------------+-----------+-----------+-----------+
    3 rows in set (0.00 sec)
    ```

    事务2：

    ```insert into test_lock values (4,14,104,4)```，等待事务1提交。

    ```update test_lock set c = 18 where a = 8```，直接成功。

    结论：非等值查询，上匹配索引的RECORD锁，上匹配范围的GAP锁（data_locks表第三行8是下一行的主键，但未锁该行，由此可推测上锁GAP为a=2至a=5、a=5至a=8的行，不包括边界行 ~~，2未显示~~ ，LOCK_MODE="X"，LOCK_DATA=5的锁为(2,5]的Next-Key Lock。

3. 等值查询与非等值查询共存
- 有匹配行
    事务1：

    ```select * from test_lock where a <= 6 and a > 3 for update```。

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   4550 | NULL       | TABLE     | IX        | NULL      |
    |   4550 | PRIMARY    | RECORD    | X         | 5         |
    |   4550 | PRIMARY    | RECORD    | X,GAP     | 8         |
    +--------+------------+-----------+-----------+-----------+
    3 rows in set (0.01 sec)
    ```

    事务2：

    ```insert into test_lock values(7, 17,107,7)```，等待事务1提交。

    ```insert into test_lock values(3, 13,103,9)```，等待事务1提交。

    结论：等值条件未匹配，与非等值查询情况相同，上锁的是索引（此处为一个开区间2-8），所有该区间的更新都会阻塞。

- 有匹配行

    事务1：

    ```select * from test_lock where a <= 8 and a > 3 for update```，有匹配行。

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+------------------------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA              |
    +--------+------------+-----------+-----------+------------------------+
    |   4551 | NULL       | TABLE     | IX        | NULL                   |
    |   4551 | PRIMARY    | RECORD    | X         | supremum pseudo-record |
    |   4551 | PRIMARY    | RECORD    | X         | 5                      |
    |   4551 | PRIMARY    | RECORD    | X         | 8                      |
    +--------+------------+-----------+-----------+------------------------+
    4 rows in set (0.01 sec)
    ```

    事务2：

    ```insert into test_lock values(9, 19,109,9)```，等待事务1提交。

    看起来不太合理，`像是个bug`。插入一条a=10的数据，会发生什么。

    此后实验数据为：
    ```
    mysql> select * from test_lock;
    +----+----+-----+----+
    | a  | b  | c   | d  |
    +----+----+-----+----+
    |  2 | 12 | 102 |  2 |
    |  5 | 15 | 105 |  5 |
    |  8 | 18 | 108 |  8 |
    | 10 | 20 | 110 | 10 |
    +----+----+-----+----+
    4 rows in set (0.01 sec)
    ```

    事务1：

    ```select * from test_lock where a <= 8 and a > 3 for update;```

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   4554 | NULL       | TABLE     | IX        | NULL      |
    |   4554 | PRIMARY    | RECORD    | X         | 5         |
    |   4554 | PRIMARY    | RECORD    | X         | 8         |
    +--------+------------+-----------+-----------+-----------+
    3 rows in set (0.01 sec)
    ```

    事务2：

    ```insert into test_lock values(9,19,109,9)```，直接成功。

    ```insert into test_lock values(7,17,107,7)```，等待事务1提交。


    结论：根据匹配条件，将等值条件与非等值条件取并集。InnoDb会在扫描行时上Next-Key Lock，扫描后不在结果集中的行/Gap释放锁，此前下一段的Gap Lock(8-$\infty$)似乎释放失败，加入a=10的行成功释放Next-Key Lock(8-10]。

    问题：下一段的Gap Lock(8-$\infty$)释放失败是bug还是有原因设计如此？

    事务1：

    ```select * from test_lock where a < 9 and a >= 2 for update;```

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4623 | NULL       | TABLE     | IX            | NULL      |
    |   4623 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 2         |
    |   4623 | PRIMARY    | RECORD    | X             | 5         |
    |   4623 | PRIMARY    | RECORD    | X             | 8         |
    |   4623 | PRIMARY    | RECORD    | X,GAP         | 10        |
    +--------+------------+-----------+---------------+-----------+
    5 rows in set (0.02 sec)
    ```

    结论：根据匹配条件，将等值条件与非等值条件取并集。对a=2的行上Record Lock。由于没有a=9的行，锁了8-10的GAP。

### 对于二级唯一索引列

1. 等值查询

- 无匹配行

    事务1：

    ```select * from test_lock where b = 17 for update;```
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   4586 | NULL       | TABLE     | IX        | NULL      |
    |   4586 | b_index    | RECORD    | X,GAP     | 18, 8     |
    +--------+------------+-----------+-----------+-----------+
    2 rows in set (0.01 sec)
    ```

    事务2：

    `insert into test_lock values(6,16,106,6)`，等待事务1提交。
    结论：再次证明锁的是索引，INDEX_NAME为b_index，未匹配上GAP锁，该GAP(15-18)的所有更新阻塞。

- 有匹配行

    事务1：

    ```select * from test_lock where b = 15 for update```，有匹配行。
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4588 | NULL       | TABLE     | IX            | NULL      |
    |   4588 | b_index    | RECORD    | X,REC_NOT_GAP | 15, 5     |
    |   4588 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         |
    +--------+------------+-----------+---------------+-----------+
    3 rows in set (0.01 sec)
    ```

    事务2：

    `insert into test_lock values(6,16,106,6)`，直接成功。

    结论：对于非主键的上锁，除了锁查询索引，也锁了对应主键的索引，因为出现了匹配行，通过主键锁防止对匹配行的更新。

- 更改

    ```update test_lock set a=17 where a=15;```
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4710 | NULL       | TABLE     | IX            | NULL      |
    |   4710 | b_index    | RECORD    | X,REC_NOT_GAP | 15, 5     |
    |   4710 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         |
    +--------+------------+-----------+---------------+-----------+
    3 rows in set (0.01 sec)
    ```

    事务2：


2. 非等值查询

- 有匹配行
    事务1：

    `select * from test_lock where b<17 and b>13 for update`
    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4598 | NULL       | TABLE     | IX            | NULL      |
    |   4598 | b_index    | RECORD    | X             | 15, 5     |
    |   4598 | b_index    | RECORD    | X             | 18, 8     |
    |   4598 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 5         |
    +--------+------------+-----------+---------------+-----------+
    4 rows in set (0.01 sec)
    ```

    事务2：

    `insert into test_lock values(6,16,106,6)`，等待事务1提交。

    `insert into test_lock values(3,13,103,3)`，等待事务1提交。

    结论：图中LOCK_MODE="X"为LOCK_DATA=n的Next-Key Locks，Record Lock为LOCK_MODE="X,REC_NOT_GAP"。所以，锁了GAP与RECORD。对b_index采取与主键上锁相同的策略，多锁了主键索引的Record Lock防止匹配行被修改。

3. 等值查询与非等值查询共存

    推理一下。  

### 对于非唯一索引列
1. 等值查询
- 无匹配行

    事务1:
    
    `select * from test_lock where d=7 for update;`

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA |
    +--------+------------+-----------+-----------+-----------+
    |   4672 | NULL       | TABLE     | IX        | NULL      |
    |   4672 | c_index    | RECORD    | X,GAP     | 8, 8      |
    +--------+------------+-----------+-----------+-----------+
    2 rows in set (0.01 sec)
    ```

    事务2:
    `insert into test_lock values (6,16,106,6);`，等待事务1

    结论：对于非唯一索引列做等值查询未匹配，锁GAP。

- 有匹配行

    事务1:
    
    `select * from test_lock where d=8 for update;`

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+-----------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA |
    +--------+------------+-----------+---------------+-----------+
    |   4674 | NULL       | TABLE     | IX            | NULL      |
    |   4674 | c_index    | RECORD    | X             | 8, 8      |
    |   4674 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 8         |
    |   4674 | c_index    | RECORD    | X,GAP         | 10, 10    |
    +--------+------------+-----------+---------------+-----------+
    4 rows in set (0.01 sec)
    ```

    事务2:

    `insert into test_lock values (6,16,106,6);`，等待事务1

    `insert into test_lock values (9,19,109,9);`，等待事务1

    ` update test_lock set d=7 where d = 10;`，等待事务1

    ` update test_lock set d=9 where d = 2;`，等待事务1

    ` update test_lock set d=8 where d = 2;`，等待事务1

    结论：对于非唯一索引列做等值查询有匹配，锁匹配行主键索引，锁查询索引匹配行和两边GAP。

    问题：为什么要锁两边GAP？

2. 非等值查询
- 无匹配行

    事务1:
    
    `select * from test_lock where d>11 for update;`

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+------------------------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA              |
    +--------+------------+-----------+-----------+------------------------+
    |   4683 | NULL       | TABLE     | IX        | NULL                   |
    |   4683 | c_index    | RECORD    | X         | supremum pseudo-record |
    +--------+------------+-----------+-----------+------------------------+
    2 rows in set (0.01 sec)
    ```

    事务2:
    
    `insert into test_lock values (11,21,121,11);`，等待事务1。

    `insert into test_lock values (9,19,109,9);`，直接成功。

    结论：对于非唯一索引列做非等值查询未匹配，锁GAP。

- 有匹配行

    事务1:
    
    `select * from test_lock where d>7 for update;`

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+---------------+------------------------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_DATA              |
    +--------+------------+-----------+---------------+------------------------+
    |   4690 | NULL       | TABLE     | IX            | NULL                   |
    |   4690 | c_index    | RECORD    | X             | supremum pseudo-record |
    |   4690 | c_index    | RECORD    | X             | 10, 10                 |
    |   4690 | c_index    | RECORD    | X             | 8, 8                   |
    |   4690 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 8                      |
    |   4690 | PRIMARY    | RECORD    | X,REC_NOT_GAP | 10                     |
    +--------+------------+-----------+---------------+------------------------+
    6 rows in set (0.01 sec)
    ```

    事务2:

    `insert into test_lock values (6,16,106,6);`，等待事务1。

    `insert into test_lock values (11,21,121,11);`，等待事务1。

    结论：对于非唯一索引列做非等值查询有匹配，锁匹配行主键索引，锁查询索引匹配行和对应GAP。


3. 等值查询与非等值查询共存

    推理一下。

### 对于无索引列
1. 任何查询（等值 / 非等值 / 等值+非等值），无匹配行or有匹配行

    事务1:

    `select * from test_lock where c=107 for update;`

    or

    `select * from test_lock where c=108 for update;`

    or

    `select * from test_lock where c>=108 for update;`

    or

    `select * from test_lock where c>999 for update;`

    ```
    mysql> select ENGINE_TRANSACTION_ID as trx_id, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_DATA from performance_schema.data_locks;
    +--------+------------+-----------+-----------+------------------------+
    | trx_id | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_DATA              |
    +--------+------------+-----------+-----------+------------------------+
    |   4645 | NULL       | TABLE     | IX        | NULL                   |
    |   4645 | PRIMARY    | RECORD    | X         | supremum pseudo-record |
    |   4645 | PRIMARY    | RECORD    | X         | 2                      |
    |   4645 | PRIMARY    | RECORD    | X         | 5                      |
    |   4645 | PRIMARY    | RECORD    | X         | 8                      |
    |   4645 | PRIMARY    | RECORD    | X         | 10                     |
    +--------+------------+-----------+-----------+------------------------+
    6 rows in set (0.01 sec)
    ```

    事务2:

    `insert into test_lock values (99,99,999,99);`，等待事务1提交。

    结论：对非唯一索引查询，锁全表。



### 问题：
1.锁行还是锁索引？

锁索引。

2.为什么在同一段gap上允许两个事务持有冲突的Gap Lock?

Gap Lock的唯一目的是防止其他事务插入间隙。





# 注
- select * from performance_schema.data_locks;
- select ENGINE_TRANSACTION_ID as trx_id, OBJECT_SCHEMA, OBJECT_NAME, INDEX_NAME, OBJECT_INSTANCE_BEGIN, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA from performance_schema.data_locks;
- SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;