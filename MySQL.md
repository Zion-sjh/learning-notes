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










##CTE结构
一、CTE 是什么？

CTE = Common Table Expression（公共表表达式）

本质就是：

给一段查询起个名字，当作临时表用

语法：

WITH 表名 AS (
    查询语句
)
SELECT * FROM 表名;
👉 举个最简单例子
WITH temp AS (
    SELECT num FROM Numbers WHERE num <= 10
)
SELECT * FROM temp;

你可以理解为：

👉 先定义一个“临时表 temp”
👉 再从 temp 里查数据

👉 和子查询的区别

你可能会问：这不就是子查询吗？

确实类似，但 CTE 更强：

对比	子查询	CTE
可读性	差	强
能复用	❌	✅
支持递归	❌	✅（重点）
二、递归 CTE（重点）

这才是考试/面试的核心。

👉 为什么需要递归？

比如这个表：

departments
------------------------
id | parent_id | name
1  | NULL      | 总公司
2  | 1         | 研发部
3  | 2         | 后端组
4  | 2         | 前端组

你要查：

“研发部的所有下级部门”

👉 这就是树结构查询

普通 SQL 很难写
👉 这时候就用 递归 CTE

三、递归 CTE 标准结构（必须背）
WITH RECURSIVE cte AS (

    -- ① 锚点（起点）
    SELECT * 
    FROM departments 
    WHERE name = '研发部'

    UNION ALL

    -- ② 递归部分（不断找子节点）
    SELECT d.*
    FROM departments d
    JOIN cte ON d.parent_id = cte.id

)
SELECT * FROM cte;
四、把它“翻译成人话”
① 锚点（Anchor）
SELECT * FROM departments WHERE name = '研发部'

👉 找起点（根节点）

② 递归部分（Recursive）
SELECT d.*
FROM departments d
JOIN cte ON d.parent_id = cte.id

👉 意思是：

找“当前节点的子节点”

③ UNION ALL（关键）

把两部分连起来：

👉 初始节点 + 不断扩展

👉 执行过程（非常重要）

数据库实际在做：

先找到：研发部
再找：研发部的子部门
再找：子部门的子部门
一直找，直到没有为止

👉 自动递归！

六、几个高频误区（考试必考）
❌ 误区1：需要写终止条件

不需要！

👉 没有子节点自然停止

❌ 误区2：必须有 LEVEL

不是必须

👉 只是辅助字段

❌ 误区3：ORDER BY 很重要

不重要

👉 只影响顺序

七、一句话记忆

递归 CTE = 锚点 + UNION ALL + 自连接

少一个就废。




































































