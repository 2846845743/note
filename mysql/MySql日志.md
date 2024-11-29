# 日志
## 为什么需要Undo Log
- 实现事务回滚，保障事务的原子性。事务处理过程中，如果出现了错误或者用户执 行了 ROLLBACK 语句，MySQL 可以利用 undo log 中的历史数据将数据恢复到事务开始之前的状态。
- 实现 MVCC（多版本并发控制）关键因素之一。MVCC 是通过 ReadView + undo log 实现的。undo log 为每条记录保存多份历史数据，MySQL 在执行快照读（普通 select 语句）的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录。

Tip:UndoLog刷盘的机制是和普通数据页一模一样的，没有区别

## 为什么需要BufferPool？
为了提高mysql执行效率
### BufferPool 里面存放什么？
MYsql加载数据以页为单位，所以BufferPool里面存放一个个页，大小为16kb每个

## 为什么需要Redo Log？
BufferPool可以有效提高执行效率，是基于内存的，但是如果断电了，里面的脏页怎么办？
于是就有了RedoLog。**Redo log主要是为了防止宕机后缓存池数据丢失**
执行过程：
1 执行sql
2 如果缓存没命中则去加载磁盘页到缓存
3 修改脏页数据
4 写RedoLog记录哪个地方做了什么修改
5 后台定期刷redolog到磁盘

> redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？

redoLog刷盘的时候对日志文件是顺序写，数据刷盘是随机写，这样redolog刷盘的效率高 
**至此， 针对为什么需要 redo log 这个问题我们有两个答案：**
实现事务的持久性，让 MySQL 有 crash-safe 的能力，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
将写操作从「随机写」变成了「顺序写」，提升 MySQL 写入磁盘的性能。

> redolog写完后直接刷盘吗？ 

不是的，redolog也有自己的缓存页，叫redolog buffer，是后续再刷盘的，刷新策略有三种，一般使用参数2，先写入内核空间，每隔一秒钟再刷新到盘

> redolog满了怎么办？

redolog是循环写，因为几分钟以前的缓存数据大有可能已经没用了，所以会删掉。如果写的速度太快，追上了循环结尾，则会阻塞mysql线程停下来先去执行落盘

## 为什么需要binlog
binlog的主要作用是用于数据备份和主从同步

binlog：二进制日志，用于存放mysql的全量数据，默认有三种格式
- 1.statement：将sql语句记录下来
- 2.row：记录每一行的数据的变化，这样方便映射，但是确定是会造成日志文件过大
- 3.mixed：混合
errorlog：错误日志。用于查看mysql系统报错
慢查询日志：默认是关闭的，需要设置一下，可以指定“超过几秒钟”的sql会记录在慢查询日志里

### 主从同步的原理

1 MySQL主节点将修改后的数据都记录在binLog，从节点会启动一个ioThread去读取binlog中的数据，然后写到自己的中继日志SQLLog里，再开启线程读取这个SqlLog映射到自己的库里面。并且mysql不会同步全量数据，如果需要第一次就同步完全，需要mysqlDump自己手动同步原有数据。

### 有了binlog，redolog还有啥鸟用？
binlog主要用于影响从库，而redolog主要用于主库恢复
### binlog何时刷盘？
默认不立刻刷盘，而是先存到binCache（在server层）

