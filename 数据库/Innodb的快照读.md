[toc]
## 简介 ##
- Innodb高并发的原因：所有普通select均为快照读，快照读使用 **MVCC** ，不加锁。
- MVCC实现RC,RR[隔离级别](https://blog.csdn.net/qq_40369829/article/details/79361814)，其余两个隔离级别和MVCC无关。

## MVCC原理 ##
### 回滚指针 ###
- 旧数据存在undo日志中，通过回滚指针查找历史版本数据、事务id标识事务的操作。
- 插入时，回滚指针为null。<br>![190528.insertrow.png](https://img-blog.csdnimg.cn/20190528120218179.png)
- 更新时。排他锁锁定行，修改前的值copy到undo中，修改值并使回滚指针指向Undo中的备份，记录redo。<br>![190528.updaterow.png](https://img-blog.csdnimg.cn/20190528125721268.png)

### 快照 ###
- 通过read view判断记录行是否可见。
- RR级别。
  - **事务开始**时，复制当前所有活跃事务到read view列表（快照）。
- RC级别。
  - **每个语句开始**时，复制当前所有活跃事务到read view列表（快照）。
- 列表中最早事务id为tmin，最晚事务id为tmax，读到的数据行上当前事务id为tid0。<br>![190528.readview.png](https://img-blog.csdnimg.cn/20190528130403385.png)

### 一致性快照 ###
- RR级别快照。可见性原则：
  - 事务开始时，创建该对象的事务已提交。
  - 对象未被删除，即使删除执行删除的事务未提交。

## 参考 ##
- [MVCC原理](https://mp.weixin.qq.com/s?__biz=MjM5NzAzMTY4NQ==&mid=2653930052&idx=1&sn=eb4cf71dc838e784af27dff2a1ca8d4b&chksm=bd3b582e8a4cd138536baa9a9b8a831f3f34c7790eed4c6f26ffecf0c332636ad896ed9f8da3&scene=21)
- [sql加锁步骤](https://www.cnblogs.com/yelbosh/p/5813865.html)