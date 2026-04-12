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



















