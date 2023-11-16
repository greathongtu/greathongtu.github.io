+++
title = 'sql 总结'
date = 2023-11-17T01:12:44+08:00
draft = true
tags = ['sql']
+++

## mysql 执行流程
SQL语句 - 查询缓存 - 解析器（语法分析 语意分析） - 优化器（逻辑优化 物理优化） - 执行器

## 为什么说 B+ 树查找行记录，最多只需1~3次磁盘IO
InnoDB 中页大小为16KB，一个键值对大概 8B+8B， 也就是一个页可以存1K个 KV pair，1K约等于1000个。 所以深度为3的 b+ 树索引可以维护 10^3 * 10^3 * 10^3 = 10 亿条记录。 正常每个节点不可能都填满了，所以一般 b+ 树高度在2到4 层。根节点在内存内，所以一般查找某一 KV 的行记录时只需1到3次磁盘IO


## InnoDB 关键特性
* Insert Buffer 升级为 Change Buffer
    对于非聚集索引的插入或更新操作，不是每次直接插入到索引页，而是先判断插入的非聚集索引页是否在缓冲池中，若在则直接插入；若不在，则先放入一个 `Insert Buffer` 对象中，然后再以一定的频率和情况进行 Insert Buffer 和辅助索引页子节点的 merge 操作。如此一来多个插入合并到一个操作中（因为在一个索引页中）。
    条件： 1. 索引是辅助索引； 2. 索引不是唯一的。
    Insert Buffer 是一个 B+ 树
* Double Write
* Adaptive Hash Index
* Async IO
* Flush Neighbor Page

## 覆盖索引
覆盖索引（covering index ，或称为索引覆盖）即从非主键索引中就能查到的记录，而不需要查询主键索引中的记录，避免了回表的产生减少了树的搜索次数，显著提升性能

## 索引下推

```sql
# 根据 key1 的索引先找大于 z 的，然后再判断a结尾，之后才回表
select * from s1 where key1 > 'z' AND key1 like '$a'
```

## EXISTS 和 IN 的区别

```sql
# EXISTS 的实现相当于外表循环： A 表小用 exists， B表小用IN
# EXISTS 先查外表； IN 先查内表
# not in 和not exists：如果查询语句使用了not in，那么内外表都进行全表扫描，没有用到索引；而not extsts的子查询依然能用到表上的索引。所以无论那个表大，用not exists都比not in要快。
select * from A where cc IN (SELECT cc from B)

select * from A where EXISTS (select cc from B where B.cc =A.cc)
```

## 范式

1NF: 每个属性要是原子性的
2NF：每一条数据记录都是唯一可标识的。所有非主键字段必须完全依赖主键
3NF：所有非主键字段不能依赖于其他非主键字段
反范式：有些冗余是必要的。业务优先
巴斯范式：更强的3NF

## ER模型
entity（矩形）-relationship（菱形）
属性（椭圆形）

一个实体转换成一个数据表，属性转换成表的字段。

使用主键和外键越多越好：证明实体之间冗余度低

## 调优的维度和步骤
1. 选择合适的DBMS
2. 优化表设计
3. 优化逻辑查询
4. 优化物理查询（索引的创建和使用）
5. 使用Redis 作为缓存
6. 库级优化（读写分离，数据分片）

## ACID
* atomic：一个事务要么commit 要么rollback
* consistency：数据从一个`合法性状态`到另一个`合法性状态`。语义上的一致性
* isolation：一个事务内部的操作和并发的其他事务是隔离的
* durability：事务提交后改变是永久的。通过事务日志保证的。包括`重做日志redo log`和`回滚日志undo log`

数据并发问题：
* 脏写：不可接受
* 脏读：Session A读了 Session B更新但没commit 的字段
* 不可重复读：Session A读了 Session B更新且commit 的字段
* 幻读：A读了一个字段，B又插入了一些新的行；这时候A读会发现多处几行

隔离级别
* read uncommited 有脏读问题
* read commited 有不可重复读问题
* repeatable read 有幻读问题
* serializable 并行太低

## MySQL 事务日志
* redo log：存储引擎层生成的日志，记录的是`物理级别`上的页修改日志，主要为了保证数据的可靠性。事务commit 的时候将redo log buffer 刷新到 redo log file
* undo log：存储引擎层生成的日志，记录的是`逻辑操作`日志。如Insert语句在undo log里要记录一条对应的Delete 操作。主要用于事务的`回滚`和`一致性非锁定读`(undo log 回滚行记录到某种特定的版本——`MVCC`)

* 为什么需要redo log：缓冲池的设计是为了协调 CPU 速度与磁盘速度的鸿沟。若每次一个页发生变化，就将新页的版本刷新到磁盘，开销太大。且若刷新时发生宕机，数据就无法恢复了。所以引入 `Write Ahead Log` 策略：事务提交时，先写redo log，再修改页。这是 ACID 中 Durability 的要求。

Checkpoint： 
    1. 缩短数据库恢复时间。
    2. 缓冲池不够用时，脏页刷新到磁盘
    3. redolog 不可用时，刷新脏页

* redo log 和 bin log 区别：redo log是存储引擎层生成的；bin log 是数据库层产生的，只有事务提交时才会一次写入到bin log中，而redo log 会一直记录

* undo log 作用：1. 回滚数据 2.MVCC

* undo log类型
    1. insert undo log：事务隔离性的要求导致commit之后可以直接删除undo log
    2. update undo log：可能需要提供MVCC机制，不能立刻删除。undo log 在事务 commit 之后，为了重用，放到链表中。purge线程判断是否删除 undo log 及 undo log 所在的页

## 并发问题的解决方法
* 方案一：读用MVCC， 写用加锁
    * 在 read committed 隔离级别下，每次select 都生成一个 ReadView
    * 在 repeatable read 隔离级别下，第一次select 时生成一个 ReadView，后续复用这个ReadView
* 方案二：读写都用锁（S lock，X lock）
```sql
SELECT ... LOCK IN SHARED MODE;
SELECT ... FOR SHARE; # S lock
SELECT ... FOR UPDATE; # X lock
``` 

## 表锁的分类
* S lock & X lock
* 意向锁（intention lock）：给某一行加上排他锁，DB会给更大一级的空间加上意向锁，告诉其他人这个数据页或数据表上有人上过排他锁了
* 自增锁（auto-inc lock）：向使用含有AUTO_INCREMENT列的表中插入数据时需要获取的一种特殊的表级锁
* 元数据锁（MDL lock）：当对表结构变更的时候要加MDL写锁，crud加MDL读锁

## 行锁的分类
* 记录锁（record lock）
* 间隙锁（Gap lock）：MySQL在repeatable read级别下解决幻读问题方法1：MVCC， 2.加锁。加锁的时候无法给尚不存在的幻影记录加锁。gap lock就是为了解决这个问题
* 临键锁（Next—Key lock）：前两种锁的结合：即锁住某条记录，又阻止其他事务在该记录前边的间隙插入新记录。在repeatable read 级别下使用
* 插入意向锁（Insert Intention Locks）：一个事务在插入记录时要判断插入位置是否有gap锁（包括next-key lock），有的话要等待，同时生成一个插入意向锁表示有事务想插入

## 页锁
* 悲观锁 乐观锁
* 显示锁 隐式锁

## 处理死锁
1. 等待直到超时
2. 死锁检测进行死锁处理

## MVCC
实现依赖于 `隐藏字段（trx_id, roll_pointer)` `Undo Log` `Read View`
是一种快照读，前提是隔离级别不是串行级别。read uncommitted也不需要MVCC。
ReadView 主要4个重要内容： creator_trx_id, trx_ids(当前活跃的读写事务), up_limit_id(活跃事务中最小的事务ID), low_limit_id(生成ReadView 时系统应该分配给下一个事务的ID)。


ReadView 判断记录的哪个版本可见：靠对比被访问版本的 trx_id 与 ReadView 中的 `creator_trx_id` `up_limit_id` `low_limit_id` `trx_ids`

ReadView 整体操作流程：
1. 首先获取事务自己的版本号，也就是事务ID
2. 获取 ReadView
3. 查询获得的数据，与 ReadView 中的事务版本号相比
4. 如果不符合 ReadView 规则，就要从 Undo Log 获取历史快照
5. 返回符合规则的结果

* 在 read committed 隔离级别下，每次select 都生成一个 ReadView
* 在 repeatable read 隔离级别下，第一次select 时生成一个 

## 日志类型
* 慢查询日志
* 通用查询日志
* 错误日志
* 二进制日志（记录所有更改数据的语句（ddl，dml，没有查询的），用于主从服务器数据同步；服务器故障恢复，相比redo log，binlog是逻辑日志，redolog是物理日志）
* 中继日志（relay log：用于主从服务器架构，从服务器用来存放主服务器bin log的一个中间文件）
* 数据定义语句日志
