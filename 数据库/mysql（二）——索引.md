
@[toc]
## 索引模型 ##
- **哈希表**：适用于只有等值查询的场景。
- **有序数组**：适合静态存储引擎的等值查询和范围查询场景。
- **N叉树**：读写近似LogN复杂度，适配磁盘访问模式。

## Innodb索引分类 ##
### 聚集索引 ###
- 按照主键构建的**B+树**，叶子节点存放整张表的行记录，称为**数据页**，默认16K。
- 每张表只能有**一个**。
- 聚集索引**逻辑上连续**，物理上不一定连续：
  - 数据页通过双向链表链接，按照主键排序。
  - 数据页中的记录通过双向链表维护，物理上不一定按照主键的顺序。

### 辅助索引（二级索引） ###
- B+树。
- 可以有多个。
- 叶子节点包含索引键值和主键，查询需要**回表**。
- 适用条件：**高选择性**（字段取值范围很广，几乎不重复）字段适合建立B+树索引。
  - Show Index 结果中的 Cardinality表示表中不重复的列的预估值，和总行数的比值接近于1，即为高选择性。

## 索引维护 ##
### 新增记录 ###
- 在记录之间插入一条记录，可能造成**页分裂**，影响插入性能和数据页利用率。
- 使用自增主键的插入模式是追加操作，不涉及页分裂，性能较高。
- 主键长度越小，普通索引的叶子节点就越小，占用的空间也就越小。

> 从**性能和存储空间**方面考量，自增主键更合理。

### 重建索引 ###
```sql
alter table T drop index k;
alter table T add index(k);
```
- 索引引可能因为删除、页分裂等原因，导致数据页有空洞。
- 重建索引的过程会创建一个新的索引，把数据按顺序插入，这样页面的利用率高（15/16），索引更紧凑、更省空间。

## 索引特征 ##
### 联合索引 ###
- 是B+树。
- 对表上多个列进行索引，叶子节点上的键值对是按照定义字段你的顺序逻辑有序的。
- **[最左前缀原则](https://time.geekbang.org/column/article/69636)**：对于联合索引(a,b,c)，应按照abc的顺序使用，等效于：a,ab,abc三个索引。

### 覆盖索引 ###
- 从辅助索引的键值中能查到需要的记录，[无需再**回表**](https://mp.weixin.qq.com/s/y0pjtNUZhOW2ZBOy4m-xsA)查聚集索引，减少树搜索次数。

### 索引下推 ###
- ICP优化 Index Condition Pushdown， 遍历索引的同时使用索引键值进行where过滤。
- **过滤放在存储引擎层**，减少sql层拉取的数据。

### 前缀索引 ###
- 主要用来存储字符串，截取字符串前缀作为索引：```alter table SUser add index index2(email(6)); ```
- 特点：节省空间，但可能降低区分度。
- 走前缀索引无法使用覆盖索引，需要回表。
- 前缀区分度不好时的替代方法，都不支持范围查询。
  - 倒序存储。无额外存储空间，cpu开销小。
  - hash存储。区分度大。

### change buffer ###
- **追加式存储**的思想，对写优化。如果非唯一的非聚集索引不在内存，不从磁盘读索引，直接在redo和change buffer记增量操作。
- 前身是insert buffer。在insert的基础的上，支持update，delete。
- 使用条件：  
  - 更新**非聚集索引**。
  - 索引**非唯一**（非Unique）。
- 优点：减少索引页不在内存中时随机读磁盘的开销。
- 缺点：
  - 大量写时宕机，导致缓冲池数据未合并，从redo恢复时间较长。
  - 大量写时占用过多的缓冲池内存，最多1/2。
- [唯一索引和普通索引对比](https://time.geekbang.org/column/article/70848)。
	- ```insert into t(id,k) values(id1,k1),(id2,k2);```一个索引在内存，一个在磁盘。

#### 写流程 ####
- ![201108.update.buffer.png](https://img-blog.csdnimg.cn/img_convert/d61b6b6171867b167027d811028b8055.png)
- Page1 在内存中，直接更新内存。
- Page2 没有在内存中，就在内存的 change buffer 区域，记录下插入行的信息。
- 将上述两个动作记入 redo log。

#### 读流程 ####
- ![201108.select.buffer.png](https://img-blog.csdnimg.cn/img_convert/baebd4419a63b7033a7e3a893a0f1f5f.png)
- 读page1直接返回。
- 读page2，从磁盘读取Page2之后Merge change buffer中的日志返回，merge过程如下：
  - 磁盘读page2.
  - 在page2上应用change buffer里相关的记录，得到新的数据页。
  - 写redo，包括数据页和change buffer的变更。

#### 合并时机 ####
- 辅助索引页被读到缓冲池，如select。
- 追踪辅助索引页的Insert Buffer Bitmap监测到辅助索引页可用空间不够。
- Master线程刷新多个索引页。


### 自适应hash索引 ###
- 从缓冲池的B+树页构建，存在缓冲池**自适应哈希索引区**，是对**页**的索引，不是整张表的。
- 条件：
  - 连续访问模式（查询条件）一致。
  - 次数超过页中记录/16。

## 索引不符预期的场景 ##
### 查询数据占比大 ###
- 需要访问的数据占比较大（20%左右），优化器会走聚集索引。如果走辅助索引，回表查询的时候是随机读，**远慢于顺序读**。

### 函数操作 ###
- 如果对**索引字段做函数操作**，会[破坏索引值的有序性](https://time.geekbang.org/column/article/74059)，优化器走聚集索引全表扫描。
- 函数操作分类：
  - 条件字段函数操作：```select count(*) from tradelog where month(t_modified)=7;```
  - 隐式类型转换：```select * from tradelog where tradeid=110717;```
    - tradeid为varchar，转成int，相当于条件字段函数操作，```select * from tradelog where  CAST(tradid AS signed int) = 110717;```
  - 隐式字符编码转换：varchar字段join操作，在**被驱动表**的索引字段上加函数操作，相当于```select * from trade_detail  where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value; ```

### 优化器权衡 ###
- 比如涉及order by操作，优化器有较大的倾向走order by字段的索引。

#### 例子 ####
- 问题：
	- order表按照uid分库分表，非唯一索引为：uid、create_time。需要查询某用户某段时间某个活动的订单，并按照创建时间倒序。
	- sql为：``` select * from order where uid = ? and activity= ? and create_time > ? and create_time < ?  order by create_time desc```, uid索引区分度较高，预期走Uid索引
	- 线上发现sql 99线较高，explain发现sql走了create_time索引，大部分扫到的数据都不是目标用户的。
- 原因：
	- 取数发现该表有热点用户，占据了约10%的数据，mysql执行计划取样认为uid索引区分度不够高，走uid索引权重不够高。
	- sql 需要根据create_time排序，走create_time取数权重更高。
- 解决：
	- 思路：修改order by 的条件，降低走create_time索引的权重，提高走uid的权重。
	- 方法：两次修改order by条件之后最终走uid索引， ```order by uid, create_time```， ```order by uid, activity_id, create_time ```

## 相关 ##
- [mysql（一）——架构和执行流程](https://blog.csdn.net/qq_40369829/article/details/100154362)
- [mysql（二）——索引](https://blog.csdn.net/qq_40369829/article/details/100154514)
- [mysql（三）——日志](https://blog.csdn.net/qq_40369829/article/details/100154560)
- [mysql（四）——快照读](https://blog.csdn.net/qq_40369829/article/details/91359489)
- [mysql（五）——锁](https://blog.csdn.net/qq_40369829/article/details/100154535)
- [mysql（六）——高可用](https://blog.csdn.net/qq_40369829/article/details/110413780)
- [mysql（七）——部分语句实现](https://blog.csdn.net/qq_40369829/article/details/110413795)