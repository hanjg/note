[toc]
## 分类 ##
- **扁平事务**。所有事务在同一层次。
- **带保存点的扁平事务**。放弃整个事务开销大，可回滚至保存点。
- **链事务**。提交一个事务后，把上下文传递给第二个事务。
- **嵌套事务**。顶层事务提交，子事务才会提交。
- **分布式事务**。

## 实现 ##
### redo ###
- redo在Innodb存储引擎产生，是物理日志，记录**页**修改操作，在事务进行过程中不断写入。
- 对比biglog:在mysql数据库上层产生，是逻辑日志，记录**sql语句**，在事务完成后一次写入。<br>![190828.binredo.png](https://img-blog.csdnimg.cn/20190828112002269.png)<br> T*为提交时的日志
- LSN（日志序列号）的含义：
  - 重做日志的写入总量。
  - checkpoint的位置。表示已经刷新到磁盘页上的LSN，恢复时从checkpoint开始。
  - 页的版本。数据库启动时，如果redo日志中的LSN大于页的版本且事务已经提交，则应用重做日志。

### undo ###
- undo是逻辑日志，记录**行**回滚到特定版本，存放在共享表空间的undo段。
- 实现rollback和 [mvcc](https://blog.csdn.net/qq_40369829/article/details/91359489) 。
- undo格式：
  - insert undo log：插入操作的undo，对事务本身可见，提交后可删除。
  - update undo log：更新和删除的undo，需要提供mvcc，提交后放入Undo链表，purge后删除。

### purge ###
- 用于最终完成delete和update。
- 实际执行的是delete不再被所有事务引用的Undo段。

## 事务的隔离级别 ##
- [https://blog.csdn.net/qq_40369829/article/details/79361814](https://blog.csdn.net/qq_40369829/article/details/79361814)