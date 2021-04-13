[toc]
## redolog ##
- 在**Innodb**存储引擎产生，环形**物理日志**，通常配置为4个1G文件。记录**页**修改操作，在事务进行过程中不断写入。<br>![201109.redo.png](https://static001.geekbang.org/resource/image/a2/e5/a25bdbbfc2cfc5d5e20690547fe7f2e5.jpg)
- **LSN**（Log Sequence Number）的含义：
  - **redo log**的写入总量。
  - **checkpoint**的位置。表示已经刷新到磁盘页上redo log位置，恢复时从checkpoint开始。
  - **页**的版本。数据库启动时，如果redo log中的LSN大于页的版本且事务已经提交，则应用重做日志。

### checkpoint ###
- checkpoint和redo日志通过LSN标识版本号，即时序。
- 每次写入长度为 length 的 redo log， LSN 的值就会加上 length。
- checkpoint之后redo日志对应的脏页均未刷新到磁盘。

### redolog写入 ###
- ![201117.redologwrite.png](https://static001.geekbang.org/resource/image/9d/d4/9d057f61d3962407f413deebc80526d4.png)
- 事务执行时，写入redo log buffer。
- 控制写入时机的参数，innodb_flush_log_at_trx_commit：
  - =0。每次提交事务，只写redo log buffer中。
    - 后台线程1s一次write到文件系统缓存并fsync到磁盘。
    - 有可能持久化未提交事务的redolog。
  - =1。每次提交事务，fsync到磁盘。
  - =2。每次提交事务，write到文件系统缓存。

#### 组提交优化 ####
- 并发事务持久化到磁盘。<br>![](https://static001.geekbang.org/resource/image/93/cc/933fdc052c6339de2aa3bf3f65b188cc.png)
- 步骤：
  - 事务1到达，为leader。
  - 事务1开始写盘，LSN增加为160。
  - 事务1写盘，将LSN小于160的一起写盘，即同时持久化事务2和3。
- redo和binlog的fsync拖到两个write之后，等待组提交。<br>![201117.groupcommt.png](https://static001.geekbang.org/resource/image/5a/28/5ae7d074c34bc5bd55c82781de670c28.png)
- 提升组提交效果，可以减少IO压力，但可能增加语句响应时间的参数：
  - binlog_group_commit_sync_delay：延迟多少微秒后才调用 fsync;
  - binlog_group_commit_sync_no_delay_count：累积多少次以后才调用 fsync。 

## binlog ##
- 在**server**层产生，是追加式的**逻辑日志**，记录**行**的修改，在事务完成后一次写入。<br>![190828.binredo.png](https://img-blog.csdnimg.cn/20190828112002269.png)<br> T*为提交时的日志

### binlog格式 ###
- statement格式。记录sql原文，可能造成主备不一致。
  - 比如delete+limit，主备不走同一个索引。
- row格式。记录主键id和操作前后的值。
- mixed。mysql判断优先使用statement，如果可能不一致则使用row。

### binlog写入 ###
- ![201117.binlogwrite.png](https://static001.geekbang.org/resource/image/9e/3e/9ed86644d5f39efb0efec595abb92e3e.png)
- 事务执行时，写入binlog cache。
  - 一个事务的binlog不能拆分，如果大小超过binlog_cache_size，临时文件暂存。
- 事务提交时，binlog cache写入binlog file，并清空cache。
- 控制写binlogfile时机的参数，sync_binlog：
  - =0。每次提交事务，只write到文件系统缓存，不fsync到磁盘。
  - =1。每次提交事务，fsync。**牺牲吞吐换取持久性**。
  - N>1。每次提交事务，write，累计N次fsync。

## redo和binlog的一致性 ##
- [两阶段提交](https://time.geekbang.org/column/article/73161)。<br>![201104.write.bin.redo.log.png](https://static001.geekbang.org/resource/image/2e/be/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)
- [崩溃恢复的规则](https://time.geekbang.org/column/article/76161)：
  - redo log完整，且已经commit，直接提交。
  - redo log只有完整的prepare，binlog完整则提交，否则回滚。判断binlog的完整性：
    - statement格式的binlog，最后会有 COMMIT。
    - row 格式的 binlog，最后会有一个 XID event。
    - redo log和binlog通过**XID**关联。

### 双1配置 ###
- sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成 1。
- 一个事务提交前，**两次刷盘**，一次是 redo log（prepare 阶段），一次是 binlog。

## undolog ##
- undo是逻辑日志，记录**行**回滚到特定版本，存放在共享表空间的undo段。
- 实现rollback和 [mvcc](https://blog.csdn.net/qq_40369829/article/details/91359489) 。
- undo格式：
  - insert undo log：插入操作的undo，对事务本身可见，提交后可删除。
  - update undo log：更新和删除的undo，需要提供mvcc，提交后放入Undo链表，purge后删除。

## 相关 ##
- [mysql（一）——架构和执行流程](https://blog.csdn.net/qq_40369829/article/details/100154362)
- [mysql（二）——索引](https://blog.csdn.net/qq_40369829/article/details/100154514)
- [mysql（三）——日志](https://blog.csdn.net/qq_40369829/article/details/100154560)
- [mysql（四）——快照读](https://blog.csdn.net/qq_40369829/article/details/91359489)
- [mysql（五）——锁](https://blog.csdn.net/qq_40369829/article/details/100154535)
- [mysql（六）——高可用](https://blog.csdn.net/qq_40369829/article/details/110413780)
- [mysql（七）——部分语句实现](https://blog.csdn.net/qq_40369829/article/details/110413795)