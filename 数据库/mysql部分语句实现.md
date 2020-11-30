[toc]
## delete ##
- innodb_file_per_table = ON，表数据存在文件中，而不是共享表空间。
- drop table可以直接删除文件。
- delete 把记录的位置标记为可复用，磁盘上文件不会变小，造成空洞。
  - 随机insert也可能造成页分裂，从而造成空洞。
- [删库恢复数据的办法](https://time.geekbang.org/column/article/78658)。

## optimize ##
### recreate ###
- ```alter table A engine=InnoDB```

#### 锁表ddl ####
- 5.5版本之前。<br>![201110.lock.ddl.png](https://static001.geekbang.org/resource/image/02/cd/02e083adaec6e1191f54992f7bc13dcd.png)
- 往临时表中插数据时业务禁写。

#### Online ddl ####
- 5.6之后。<br>![201110.online.ddl.png](https://static001.geekbang.org/resource/image/2d/f0/2d1cfbbeb013b851a56390d38b5321f0.png)
- 步骤：	
	- 建立一个临时文件，扫描表 A 主键的所有数据页；
	- 用数据页中表 A 的记录生成 B+ 树，存储到临时文件tmp中；
	- 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中，对应的是图中 state2 的状态；
	- 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同的数据文件，对应的就是图中 state3 的状态；
	- 用临时文件替换表 A 的数据文件。  
- [online ddl详细步骤](https://www.cnblogs.com/cchust/p/4639397.html)
  - MDL写锁在copy数据之前退化成读锁，不阻塞读写，禁止其他DDL。
- 推荐使用开源的gh-ost。

### 对比 ###
- recreate：重建表。
  - 重建表时，InnoDB每个页留了 1/16 给后续的更新用，重建表之后不是最紧凑的。
- analyze table：对表的索引信息做重新统计，没有修改数据，加MDL读锁。
- optimize table ： recreate+analyze。

## count ##
- [count(*)这么慢，我该怎么办](https://time.geekbang.org/column/article/72775)。

### 几种计数方式 ###
- MyISAM把一个表的总行数存在了磁盘上，执行时直接返回这个数。不支持事务。
- Innodb需要一行行从引擎中读数据然后累加。性能较差。
  - 支持事务，有多个版本，执行时才能确定视图和可见记录。
  - 走最小的索引遍历。
- show table status：估算值，不准确。
- 新增计数表：单库事务保证一致性。

### count原则 ###
- server 层要什么就给什么；
- InnoDB 只给必要的值；
- 现在的优化器只优化了 count(*) 的语义为“取行数”，其他“显而易见”的优化并没有做。

### count函数对比 ###
- count(字段)：
  - InnoDB遍历整张表，读出字段。
  - server 层拿到字段后，如果字段not null，直接累加，否则判断非null之后累加。
- count(主键id)：
  - InnoDB遍历整张表，把每一行的id值都取出来，返回给 server 层。
  - server 层拿到 id 后直接按行累加。
- count(1)：
  - InnoDB遍历整张表，不取值，对每一行返回1。。
  - server 层拿到 id 后直接按行累加。
- count(*)：专门优化，同count(1)按行累加。

## order by ##
- [“order by”是怎么工作的](https://time.geekbang.org/column/article/73479)
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;

select city,name,age from t where city='杭州' order by name limit 1000  ;
```

### 全字段排序 ###
- ![201110.allsort.png](https://static001.geekbang.org/resource/image/6c/72/6c821828cddf46670f9d56e126e3e772.jpg)
- 步骤：
  - 索引上查满足条件记录的主键。
  - 回表查需要返回给客户端的列，写sort_buffer。
  - 内存排序。如果需要的内存超过sort_buffer_size，使用临时文件归并排序。
  - 返回前n行。

### rowid 排序 ###
- ![201110.rowidsort.png](https://static001.geekbang.org/resource/image/dc/6d/dc92b67721171206a302eb679c83e86d.jpg)
- 数据行过长，超过max_length_for_sort_data，只把需要排序的列和主键放入sort_buffer排序。
- 步骤：
  - 索引上查找满足条件记录的主键。
  - 回表查需要排序的列和主键，写sort_buffer。
  - 内存排序。如果需要的内存超过sort_buffer_size，使用临时文件归并排序。
  - 前n行用主键id回表查需求的列。
  - 返回客户端。

> 内存够用则尽量使用内存，减少磁盘访问。<br>
> rowid排序多一次回表，可能多读一次磁盘，mysql优先全字段排序。

### 不排序 ###
- 原来的**数据无序**，所以MySQL需要生成**临时表**，并且在临时表上做排序操作。
- 如果有(city,name)的索引，可以保证where过滤后数据有序，直接遍历得到需要的记录主键，回表得到结果。
- 如果有(city,name,age)的索引，可以覆盖索引免回表。

## join ##
- 如果能使用被驱动表索引，可以join，否则尽量不用Join。
- join时用小表（按照各自条件过滤，参与join的总数据量少）做驱动表。
- [join分类](https://time.geekbang.org/column/article/79700)，[join优化](https://time.geekbang.org/column/article/80147)。

### Index Nested-Loop Join ###
- 可以使用被驱动表t2的索引。
- 扫驱动表t1全表，逐条走**t2索引**查找(BKA算法利用join_buffer优化，批量查t2)。<br>![201119.nlj.png](https://static001.geekbang.org/resource/image/d8/f6/d83ad1cbd6118603be795b26d38f8df6.jpg)

### Block Nested-Loop Join ###
- 被驱动表t2上没有可用索引。
- join_buffer暂存。
  - t1读入joinbuffer。
  - **扫t2**，和joinbuffer中的数据逐一对比，满足条件的作为结果集。<br>![201119.bnl1.png](https://static001.geekbang.org/resource/image/15/73/15ae4f17c46bf71e8349a8f2ef70d573.jpg)
- 如果join_buffer超过join_buffer_size，默认256k，**分段放t1**，**扫t2**和joinbuffer对比出结果集。<br>![201119.bnl2.png](https://static001.geekbang.org/resource/image/69/c4/695adf810fcdb07e393467bcfd2f6ac4.jpg)
- 对系统的影响：
  - 由于会多次扫描t2，io压力。
  - 判断join需要M*N次对比，cpu压力
  - 如果扫描t2持续超过1s，会造成t2都进入young区，导致buffer pool正常业务命中率降低。
- 可通过[临时表](https://time.geekbang.org/column/article/80449)加索引触发BKA优化。
  - 一个线程一个临时表，线程退出时回收。

## group by ##
### 基本流程 ###
- ![](https://static001.geekbang.org/resource/image/3d/98/3d1cb94589b6b3c4bb57b0bdfa385d98.png)
- 创建内存临时表（如果内存不够磁盘辅助），包含m,c，**主键m**。
- 扫索引a，根据id%10，在临时表中累计c。
- 根据m排序。<br>![201119.groupby.png](https://static001.geekbang.org/resource/image/03/54/0399382169faf50fc1b354099af71954.jpg)

### 使用原则 ###
- 对group by没有排序要求，加上group by null。
- group by尽量使用索引，由于数据有序可边读边输出结果，避免临时表和排序。
- group by尽量使用内存临时表，tmp_table_size调整内存临时表上限。
- 如果数据量太大，直接[SQL_BIG_RESULT](https://time.geekbang.org/column/article/80477)使用磁盘临时表。
