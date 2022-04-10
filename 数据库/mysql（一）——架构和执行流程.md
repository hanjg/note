[toc]
## mysql架构 ##
- ![190805.mysql.png](https://img-blog.csdnimg.cn/20190805101435768.png)
### server层 ###
- **连接器**：负责跟客户端建立连接、获取权限、维持和管理连接。
  - 临时使用的内存在连接对象中，连接断开时释放，建议定期断开长连接。
  - 连接方式：
    - 命名管道、共享内存、UNIX域套接字。需在一台服务器。
    - TCP/IP，最多。
- **查询缓存**：执行过的语句及其结果k-v形式保存在内存中。
  - 查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。
  - mysql8.0废弃。
- **分析器**：词法分析。
- **优化器**：决定走哪个索引、join的表连接顺序。
  - [优化器有可能选错索引](https://time.geekbang.org/column/article/71173)，上线前最好explain sql。
- **执行器**：check表权限，循环操作底层引擎层读写。

### 引擎层 ###
- **插件式**基于**表**的存储引擎管理底层文件。
- **Innodb**。支持事务，行锁设计，支持外键，默认读不加锁。面向在线事务处理（OLTP）。
- **MyISAM**。不支持事务，支持全文索引。面向在线分析（OLAP）。
- [Memory](https://time.geekbang.org/column/article/80495)。堆组织表，数据单独存放，索引上保存数据位置。适合做临时表。
  - 内存读写快，支持hash索引。
  - 只有表级锁，数据无持久化，备库重启会删除主库的内存表。

### 存储层 ###
- 逻辑存储结构：**表空间**（tablespace），**段**（segment），**区**（extent,1MB），**页/段**（page/block，默认16KB），行。

### 特点 ###
- 优点：
  - 关系型数据库，支持复杂条件查询。
  - 性能较高。
  - 成熟的多活、主从、sharding方案。
- 缺点：
  - 大量数据需要分库分表。需要中间件，拆库迁移数据较麻烦。

## Innodb引擎架构 ##
- ![190805.innodb.png](https://img-blog.csdnimg.cn/20190805102907958.png)

### 后台线程 ###
- Master Thread： 异步刷新缓冲池数据到磁盘，保持数据一致性。包括脏页刷新、合并插入缓冲、UNDO页回收等。
- IO Thread： 处理AIO请求的回调。
- Purge Thread： 事务提交后，回收不再被事务使用的Undo页。减轻对master中业务请求的影响。
- Page Cleaner Thread: 刷新脏页。减轻对master中业务请求的影响。

### 内存池 ###
- ![190805.memory.png](https://img-blog.csdnimg.cn/20190805103941405.png)

#### 缓冲池 ####
- 读取时先读缓冲池，没有则从磁盘读取之后写缓冲池。
- 修改时修改缓冲池中的页，在checkpoint时刷新到磁盘。
- [改进LRU算法](https://time.geekbang.org/column/article/79407)管理缓冲池，新读到的页放到LRU列表的 **midpoint** 。防止扫描操作用非热点数据将热点数据从LRU列表中移除，影响缓冲命中率。
  - midpoint将链表分为靠头5/8的young，和靠尾3/8的old。
  - 访问的数据页在young，移到链表头部。
  - 访问的数据页在old：
    - 在链表中存在超过1s，移到链表头部。
    - 在链表中存在少于1s，位置不变，innodb_old_blocks_time控制时间。
  - 访问的数据页不在内存。淘汰链表尾部，插入old头部。

#### 重做日志缓冲 ####
- 在以下三种情况下刷新到磁盘重做日志：
  - Master线程每秒刷新。
  - 事务提交时刷新。
  - 重做日志缓冲池剩余空间小于一半。

#### 额外内存池 ####
- 存一些缓冲控制对象，空间不够会从缓冲池中申请。

## sql执行流程 ##
- [流程](https://time.geekbang.org/column/article/68319)：<br>![201104.mysql.logic.arch.png](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)

### 查询 ###
- [查一行记录也很慢的可能原因](https://time.geekbang.org/column/article/74687)：
  - [等锁](https://blog.csdn.net/qq_40369829/article/details/100154535)。  
	- 等MDL锁：如另一个session持有mdl写锁。
    - 等flush：flush table操作被其他查询阻塞。
    - 等行锁：当前读加锁，等其他session释放行锁。
  - 全表扫描：查询条件没有索引。
  - [历史版本过多](https://blog.csdn.net/qq_40369829/article/details/91359489)：快照读需要一个个遍历到可见的版本。<br>![201112.version.png](https://static001.geekbang.org/resource/image/46/8c/46bb9f5e27854678bfcaeaf0c3b8a98c.png)

### 修改 ###
- 写redolog、binlog、undolog，修改缓冲池数据页，刷脏页。
- [log一致性](https://blog.csdn.net/qq_40369829/article/details/100154560)。

#### WAL ####
- **Write Ahead Log**策略：
  1. 事务提交前先写**redo log**和**undo log**和**binlog**。
  2. 再修改**缓冲池中的页**，修改之后的页为脏页。
  3. 某个时机**刷新脏页到磁盘**（两次写）,并修改checkpoint位置。
  4. 如果宕机丢数据可由checkpoint之后的redo日志恢复，符合持久性要求，并减少恢复时间。
- 顺序写+组提交，持久化到磁盘的性能好。

#### 刷脏页 ####
- 可能导致业务sql rt抖一下。
- 刷脏页时机：
  - mysql**正常关闭**前。刷新所有脏页到磁盘。
  - mysql**正常运行**时。master或者Page Cleaner刷脏页到磁盘。
  - **redo log写满**，需要flush循环利用重做日志。此时，更新会被阻塞。<br>![201109.redo.png](https://static001.geekbang.org/resource/image/a2/e5/a25bdbbfc2cfc5d5e20690547fe7f2e5.jpg)
  - **LRU列表页不足**，从尾部淘汰的如果是脏页则写到磁盘。如果淘汰的脏页太多，查询rt会变长。
- 刷脏页速度：
  - innodb_io_capacity：磁盘全力刷脏页的能力，推荐为磁盘的IOPS。
  - 脏页比例：Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total，推荐在75%以下。
  - [刷脏页速度策略](https://time.geekbang.org/column/article/71806)：脏页比例增加、redo和checkpoint差距增加->刷脏页速度增加。
- 刷脏页时double write避免损坏数据页。

## Innodb特性 ##
### 两次写 ###
- 原因：
  - **部分写失效**：将缓冲池中的页写到表中，进行到一半数据库宕机，磁盘数据页的数据损坏。<br>
  - redo日志只记录对表的操作，重做需要页的原始副本，对已经损坏的页没有意义。
- 流程：
  - **刷脏页**时，先写内存中的doublewrite buffer（默认128个数据页，每个16K）。
  - 再通过buffer先后写到共享表空间**脏页副本**（顺序写）和**磁盘**（随机写）。
  - 崩溃恢复时，直接使用数据页的共享表空间副本还原原始数据页，再应用Redo log。<br> ![190816.double.png](https://img-blog.csdnimg.cn/20190816125700861.png)

### 异步IO ###
- 或者叫并发IO，索引扫描、刷新脏页等。

### 刷新临近页 ###
- innodb_flush_neighbors：刷新脏页时，如果同区有脏页，一起刷新。
- IOPS较低的机械硬盘推荐使用。
- mysql 8 默认不开启。

## 事务 ##
### 事务分类 ###
- **扁平事务**。所有事务在同一层次。
- **带保存点的扁平事务**。放弃整个事务开销大，可回滚至保存点。
- **链事务**。提交一个事务后，把上下文传递给第二个事务。
- **嵌套事务**。顶层事务提交，子事务才会提交。
- **分布式事务**。

### 长事务 ###
- 后果：
  - 回滚段占用过多空间。
  - 占用锁资源，影响其他DML执行。
  - binlog长，主从延迟。
- [措施](https://time.geekbang.org/column/article/69236)：
  - 写sql时考虑更新行数，必要时explain之后上线
  - 尽量set autocommit = 1自动提交。
  - 少用或不用只读事务。
  - 通过 SET MAX_EXECUTION_TIME 设置最长执行时间。

## 自增id ##
### 自增主键 ###
#### 存储 ####
- MyISAM：数据文件。
- Innodb：
  - 5.7和之前存在内存中，每次启动时找最大主键值然后+1。
  - 8.0之后存在redo中，重启是从redo恢复。

#### 修改 ####
- 插入数据是0，null，未指定时，当前表的AUTO_INCREMENT填充。
- 如果插入值<AUTO_INCREMENT,AUTO_INCREMENT不变，否则AUTO_INCREMENT修改为新值，默认+1。
- 自增id**递增但不连续**：
  - 事务回滚，自增id不回退。先申请id的事务回滚之后，如果id回退，之后的insert会主键冲突。
  - 批量插入语句，申请id数量递增，可能用不完。
- 自增锁加锁策略。innodb_autoinc_lock_mode。
  - 0。语句执行释放锁。
  - 1。默认值
    - 普通insert，申请id之后释放。
    - ```insert..select```之类的批量插入，语句结束之后释放。binlog_format=statement的时候，[主从一致性考虑](https://time.geekbang.org/column/article/80531)。
  - 2。申请id之后释放。
- 自增到最大值之后不变，下一次insert主键冲突。

### row_id ###
- 无主键的表系统指定。
- 范围为0~2^48-1，循环使用，理论上可能覆盖之前的记录。

### 事务Id ###
- innodb判断事务可见性使用。
- 修改操作时分配，只读事务不分配。
  - 只读事务不影响可见性，减少活跃事务数组的大小。
  - 减少申请事务Id次数和申请锁冲突。
- 事务id到2^48-1之后从0开始，[会有脏读问题](https://time.geekbang.org/column/article/83183)。不过生产中考虑写入的qps，基本不会遇到。
  - 事务B的事务id小于事务A的低水位造成。<br>![201125.dirty.png](https://static001.geekbang.org/resource/image/13/c0/13735f955a437a848895787bf9c723c0.png)

### 线程id ###
- 4个字节的数组，全局唯一分配。
- 循环使用

## 相关 ##
- [mysql（一）——架构和执行流程](https://blog.csdn.net/qq_40369829/article/details/100154362)
- [mysql（二）——索引](https://blog.csdn.net/qq_40369829/article/details/100154514)
- [mysql（三）——日志](https://blog.csdn.net/qq_40369829/article/details/100154560)
- [mysql（四）——快照读](https://blog.csdn.net/qq_40369829/article/details/91359489)
- [mysql（五）——锁](https://blog.csdn.net/qq_40369829/article/details/100154535)
- [mysql（六）——高可用](https://blog.csdn.net/qq_40369829/article/details/110413780)
- [mysql（七）——部分语句实现](https://blog.csdn.net/qq_40369829/article/details/110413795)