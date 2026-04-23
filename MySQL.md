##锁机制

primary key 会让MySQL自动给id列创建唯一的聚簇索引

SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;设置当前事务级别为RC

在 Read Committed 下，InnoDB 默认不使用 Gap Lock / Next-Key Lock 来防幻读（RC下，默认不使用 Gap Lock ，但仍可能使用）


  
#行级锁

分为Record,Gap Lock,Next-key Lock。

InnoDB在进行范围查询或索引扫描时，会对“记录+间隙”加锁，从而阻止其他事务在查询范围内插入新数据，避免幻读。

唯一索引+已存在记录+等值匹配-->优化为行锁

唯一索引+不存在记录+等值匹配-->优化为间隙锁

普通索引+等值查询+不满足需求-->向右遍历最后一个不满足的值-->next-key退化为gap key(只是间隙，不包括值)

唯一索引+范围查询-->访问到不满足条件的第一个值为止
    
不通过索引条件检索数据-->（对扫描到的每一行都加锁（全表加行锁）但仍是行锁）即行锁 + 全表扫描。InnoDB没有“行锁升级为表锁”这个机制






##InnoDB存储引擎

System Tablesapce: Change Buffer的存储区域，还可能包含表和存储索引。所有数据都在 ibdata1。

System Tablesapce是共享空间表A删了数据，空间可以给表B用，而且文件不会缩小
所以当DELETE 删除数据后，磁盘空间不会立即释放（大多数情况下）
DELETE 之后：
数据被标记为“已删除” ✔
空间 不会归还给操作系统 ❌
只能被 InnoDB 内部复用 ✔

File-Per-Table Tablespace: 存表数据（data）,聚簇索引,二级索引。每张表一个 .ibd 文件。
当DELETE 删除数据后，磁盘空间不会立即释放。因为 DELETE 本质是标记删除不是物理删除文件。

在 InnoDB 中：DELETE 操作只是标记删除，数据页中的空间变为“可复用”，不会立即归还给操作系统
OPTIMIZE TABLE table_name;ALTER TABLE table_name ENGINE=InnoDB;DROP TABLE可以删除

重做日志缓冲（redo log buffer）在内存中

重做日志文件（redo log file）在磁盘中


#MVCC

MVCC依赖于三个隐式字段，undo log日志，readview

在RR隔离级别下，InnoDB通过MVCC保证已读取数据的一致性，通过next-key lock锁住查询范围，阻止其他事务在该范围插入新数据，从未避免幻读。

redo log 在事务执行中，数据修改时实时生成。事务提交时redo log进行刷盘。保证已提交事务不会丢失

MySQL 重启后会执行：
👉 crash recovery（崩溃恢复）
流程：
扫描 redo log
找到已提交但未刷盘的数据页修改
重新执行这些修改（redo）
恢复数据一致性

undo log回滚日志 在数据修改之前生成。用于事务回滚和MVCC

redo 保证持久性，undo 保证原子性 + 隔离性

真正保证不丢数据的是：commit 时 redo log 已经持久化到磁盘

WAL预写日志：先写日志（redo log buffer → redo log file）再写数据页（Buffer Pool → 磁盘）保证崩溃后可以“重做”已提交的操作

一条 UPDATE 语句在 InnoDB 中的执行流程
1️⃣ 定位数据页（可能从磁盘加载到 Buffer Pool）
2️⃣ 修改数据页（在 Buffer Pool 中）
3️⃣ 生成 undo log（用于回滚 / MVCC）
4️⃣ 写 redo log buffer（记录物理修改）
5️⃣ commit 时：
redo log 刷盘（WAL）
6️⃣ 后续：
后台线程将脏页刷盘（不是立刻）


Read View 核心 4 个字段：
m_ids       ：当前活跃事务ID列表
min_trx_id  ：最小活跃事务ID
max_trx_id  ：下一个将要分配的事务ID
creator_trx_id ：创建该 Read View 的事务ID

为什么 MVCC 能实现“非阻塞读”：读不加锁，读的是“历史版本”。SELECT 不会阻塞 UPDATE，UPDATE 也不会阻塞 SELECT（快照读）






























































