[toc]
## MVCC简介 ##
- Innodb高并发的原因：所有普通select均为快照读，快照读使用 **MVCC** ，不加锁。
- MVCC实现mysql中的RC,RR[隔离级别](https://blog.csdn.net/qq_40369829/article/details/79361814)，其余两个隔离级别和MVCC无关。

## MVCC实现 ##
### 数据多版本 ###
- 数据历史版本存在undo日志中：
  - 通过**事务id**标识是哪个事务的操作。
  - 通过**回滚指针**串联历史版本数据。
- 插入时：回滚指针为null。<br>![190528.insertrow.png](https://img-blog.csdnimg.cn/20190528120218179.png)
- 更新时：排他锁**锁定行**，修改前的值copy到undo中，修改当前值并使回滚指针指向Undo中的备份。
- 多次更新时：比如一个值从1改成2->3->4，回滚段串成到最老版本的链表，快照读时从当前值4开始寻找可见版本。<br>![201104.update.png](https://static001.geekbang.org/resource/image/d9/ee/d9c313809e5ac148fc39feff532f0fee.png)
- 系统里没有比这个回滚段更早的read-view时：回滚段被删除。

### 可见性 ###
- 查询逻辑：普通select为**快照读**，一个数据版本，对于一个事务来说的可见性如下
  - 可见：**自己更新**的版本、在**视图创建前提交**的版本。
  - 不可见：未提交的版本、在视图创建后提交的版本。
- 更新逻辑：更新数据是先**当前读**，后写。[当前读需要加锁](https://time.geekbang.org/column/article/70562)：包括update，select + for update，select + lock in share mode

### 可见性实现 ###
- 核心思路：一致性视图和当前事务id比较。
- **一致性视图**：视图数组+高水位。每个事务维护一个视图数组，保存**创建视图时**活跃事务id。
  - **低水位**（最小**未提交**事务）：视图数组中事务id最小值。
  - **高水位**（最小**未开始**事务）：系统已经创建过的事务最大值+1。<br>![201129.view.png](https://static001.geekbang.org/resource/image/88/5e/882114aaf55861832b4270d44507695e.png)
- 视图数组创建时机：
  - RR级别：**第一个查询语句**开始时，当前所有**活跃**事务（**启动还未提交**）。
  - RC级别：**每个查询语句**开始时，当前所有活跃事务。
- 数据版本事务id的状态和可见性：
  - [0，低水位)：已提交事务，可见。
  - [低水位，高水位)：数据版本的事务id在视图数组中，未提交，不可见；否则可见。
  - [高水位，+∞)：将来启动的事务，不可见。

### 隔离级别和视图 ###
- RC。每一条语句执行前重新计算一致性视图。
- RR。事务开始时创建一致性视图。

## 参考 ##
- [MVCC原理](https://mp.weixin.qq.com/s?__biz=MjM5NzAzMTY4NQ==&mid=2653930052&idx=1&sn=eb4cf71dc838e784af27dff2a1ca8d4b&chksm=bd3b582e8a4cd138536baa9a9b8a831f3f34c7790eed4c6f26ffecf0c332636ad896ed9f8da3&scene=21)
- [sql加锁步骤](https://www.cnblogs.com/yelbosh/p/5813865.html)
- [事务隔离：为什么你改了我还看不见？](https://time.geekbang.org/column/article/68963)

## 相关 ##
- [mysql（一）——架构和执行流程](https://blog.csdn.net/qq_40369829/article/details/100154362)
- [mysql（二）——索引](https://blog.csdn.net/qq_40369829/article/details/100154514)
- [mysql（三）——日志](https://blog.csdn.net/qq_40369829/article/details/100154560)
- [mysql（四）——快照读](https://blog.csdn.net/qq_40369829/article/details/91359489)
- [mysql（五）——锁](https://blog.csdn.net/qq_40369829/article/details/100154535)
- [mysql（六）——高可用](https://blog.csdn.net/qq_40369829/article/details/110413780)
- [mysql（七）——部分语句实现](https://blog.csdn.net/qq_40369829/article/details/110413795)