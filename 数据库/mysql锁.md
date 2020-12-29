[toc]
## mysql锁的分类 ##
### 全局锁 ###
- Flush tables with read lock (FTWRL)，库只读，通常用于全库逻辑备份。
  - 如果引擎支持一致性读，mysqldump+ single-transaction，导数据之前就会启动一个事务，来确保拿到一致性视图。
- set global readonly=true 也可以全库只读，但是缺点如下：
  - readonly有可能被用来判断是从库还是主库。
  - 客户端异常，FTWRL会释放锁，readonly还是保持只读状态。

### 表级锁 ###
- **表锁**。lock tables … read/write，标识加写锁或者读锁。unlocks tables解锁。
- **元数据锁**（MDL）。读写一个表时自动加读锁，DDL变更时加写锁。
  - DDL变更会加写锁，阻塞业务语句，如果DDL被长事务阻塞还可能导致死锁。<br>![201105.mdl.png](https://static001.geekbang.org/resource/image/7c/ce/7cf6a3bf90d72d1f0fc156ececdfb0ce.jpg)
  - DDL变更时可以考虑暂停长事务。
  - 如果从库mysql dump占读锁，DDL从主库同步加写锁被阻塞，[会造成主从延迟](https://time.geekbang.org/column/article/70215)。

## Innodb锁的分类 ##
- Innodb的锁分为两种：
  1. 轻量级的闩锁（**latch**），保护内存中的数据结构；
  2. 锁（**lock**），保护数据，一般仅在commit或rollback后释放。<br>![190823.latch.png](https://img-blog.csdnimg.cn/20190823114423397.png)
- lock的分类，SS兼容，其余互斥：
  - 共享锁（S Lock），允许事务读一行记录。
  - 排它锁（X Lock），允许事务删除或更新一条记录。
- 意向锁：表级锁，**不与行级共享排他锁互斥**。事务获取Lock之前，需要获取表上的[意向锁](https://juejin.cn/post/6856780646665158670)。
  - 意向共享锁：事务希望获得某几行的共享锁。
  - 意向排它锁：事务希望获得某几行的排它锁。 <br>![190823.ISIX.png](https://img-blog.csdnimg.cn/20190823120335417.png)

## 读取加锁 ##
### 一致性非锁定读 ###
- 如果读取的行由于update或delete，加X锁，不会等锁释放，而是读行的快照（undo段实现）。[Innodb快照读](https://blog.csdn.net/qq_40369829/article/details/91359489)。

### 一致性锁定读 ###
- 对select语句显示加锁。
  - 对读取的行加**X锁**：select ... for update。
  - 对读取的行加**S锁**：select ... lock in share mode。

## 外键和锁 ##
- 外键列，如果没有显式的加索引，innodb会自动加索引，避免表锁。
- 外键值插入时：需要查询父表的记录，**用一致性锁定读**，加S锁。
- 尽量在业务层保证数据的一致性，减小数据的压力。

## 两阶段锁 ##
- **两阶段锁协议**：锁是在需要的时候才加上的，等到事务提交时才释放。
- 如果事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

### Record Lock ###
- 锁单行上的记录。
- RC级别使用。

### Gap Lock ###
- 间隙锁，锁小于记录值不包括记录本身的范围。

### Next-Key Lock ###
- 锁**范围**，**包括记录本身**。先加gap lock，再加record lock。
- RR级别使用，可解决[幻读](https://blog.csdn.net/qq_40369829/article/details/79361814)问题。

### Next-Key Lock加锁规则 ###
- [加锁规则](https://time.geekbang.org/column/article/75659)可以概括为：两个原则、两个优化和一个bug:
  - 原则1:加锁的基本单位是**索引区间，前开后闭**。
  - 原则2:查找过程中**访问到的对象才会加锁**。
  - 优化1:索引上的**等值**查询，给**唯一索引**加锁的时，next-key lock退化成**行锁**。
  - 优化2:索引上的**等值**查询，给**非唯一索引**加锁时，向右遍历时且**锁区间最后一个值不满足等值条件**的时候，next-key lock退化为**间隙锁**。
  - 1个bug:**唯一索引上的范围查询**会访问到不满足条件的第一个值为止,这个值由于原则2会加锁。
- [更多样例](https://time.geekbang.org/column/article/78427)。

#### 例1 ####
```sql
Table t （
  a int,
  b int,
  primary key(a),
  key(b)
）
```

- 如果查询条件无索引，执行```select * from t where b = 3 for update```，走聚集索引**全表扫描**，所有记录加锁。
- 表中有两条索引，(1,1),(3,1),(5,3),(7,6)这几条记录。
  - b列辅助索引，加next-key lock，索引分为几段加锁区间：(-∞,1],(1,3],(3,6],(6,+∞)。
- 执行 ```select * from t where b = 3 for update```，会对两条**索引分别加锁**。
  - 辅助索引：加Next-key Lock，并对下一个键值加gap lock，锁定辅助索引b的范围(1,3]和(3,6)。
  - 聚集索引：由于查询未使用**覆盖索引**，对a=5聚集索引加Record Lock。
- 执行``` select * from t where b > 2 for update```。
  - 辅助索引：锁住b[2,+∞)。
  - 聚集索引：由于查询未使用**覆盖索引**，加锁的辅助索引值对应的聚集索引都加锁。

#### 例2 ####
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

- ![201115.nextkeylock1.png](https://static001.geekbang.org/resource/image/3a/1e/3a7578e104612a188a2d574eaa3bd81e.png)
- 加锁步骤：
  - 定位右边界。从c=20扫到c=25，索引c加锁(15,20]和(20,25)。
  - 扫描至左边界。扫描至c=10，索引c加锁(10,15]和(5,10]。
  - 由于无法使用覆盖索引，需要回表，聚集索引上id=15,20都需要加行锁。

#### 例3 ####
- insert如果出现唯一键冲突，会在冲突的唯一值上加共享的next-key lock，需要尽快提交或回滚事务，否则锁的时间过长。
- [加S锁是为了避免这一行被别的事务删掉](https://time.geekbang.org/column/article/80801)。<br>![201124.insertblock.png](https://static001.geekbang.org/resource/image/83/ca/83fb2d877932941b230d6b5be8cca6ca.png)

## 死锁 ##
### 例1 ###
- 两个事务互相等待对方释放资源。聚集索引上id=1,2上的写锁互相等待。<br>![201105.lock.png](https://static001.geekbang.org/resource/image/4d/52/4d0eeec7b136371b79248a0aed005a52.jpg)
- 退出条件：
  - 锁超时。innodb_lock_wait_timeout 默认50s。
  - 主动死锁检测。innodb_deadlock_detect 默认on，但是n^2算法，消耗cpu，热点行更新时更为明显，会降低TPS。

### 例2 ###
- [c为唯一索引](https://time.geekbang.org/column/article/80801)。<br>![201124.deadlock.png](https://static001.geekbang.org/resource/image/63/2d/63658eb26e7a03b49f123fceed94cd2d.png)
- 加锁步骤：
  - T1：sessionA，c=5上加行锁（优化1唯一索引退化）
  - T2：sessionB和C，先加（0,5）读锁，再c=5加行锁时阻塞。
  - T3：A回滚后，B和C互相等待加写锁。