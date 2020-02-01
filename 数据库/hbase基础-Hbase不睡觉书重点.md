[toc]
## 简介 ##
### 使用场景 ###
- 数据量超过千万。
- 数据分析需求弱，复杂查询少。
- 实时性要求不高。

### 对比关系型数据库 ###
- 关系型数据库：行的各个列都是不可分割的，存储在一起。
- Hbase：行是抽象的概念，每一列是离散的，不同列可在不同机器上。

### CP ###
- 强一致：对于每一个region同时只有一个region server提供服务。
- 牺牲可用性：region server宕机，迁到其他region server，新region需要根据WAL来redo，这期间region不可用的，从而提高一致性。

## 存储架构 ##
- ![191009.row.png](https://img-blog.csdnimg.cn/20191009121551676.png)
- **namespace**:
- **table**：创建时定义列族。
- **rowkey**: 不重复的字符串，决定行存储顺序(字典序)的唯一凭证。
- **列族**：建表时确定，过期时间、缓存块大小等都是定义在列族上的，同一列族的列会尽量放在一台服务器上。
- **列**：字段。
- **单元格**：一个列的一个**版本**的值，版本号默认为写入的时间戳。

## 部署架构 ##
### 集群 ###
- ![191016.regionserver.png](https://img-blog.csdnimg.cn/20191017000056400.png)
- **region server** 是**region** 的容器。
- **region** ，一段数据的集合,或者说多个行的集合：
  - 不能跨服务器，一个region server上有多个region。
  - 在数据量大的时候会分裂，负载均衡是也会在region server之间迁移。
  - 基于HDFS，数据存取操作基于HDFS客户端接口。
  - 用预拆分初始化和自动拆分管理region，大量删除数据后用online_merge合并region。
- **master** 负责**跨region server** 的操作，如建表、移动region、合并region等。
- **zk** 管理所有的region server，包括meta节点的地址。
  - client和zk通信后直连region server，降低对master的依赖。

### region server ###
- ![191022.regionserver.png](https://img-blog.csdnimg.cn/20191022125910428.png)
- WAL：Write-Ahead Log，存在**HDFS**上。 
  - 环状滚动日志：写入效果最高、空间不会变大。
  - 滚动条件：1、WAL所在的block快满；2、WAL空间大于block的阈值。
  - WAL创建在```/hbase/.log```下，归档到```/hbase/.oldlogs```下。
  - 删除条件：当WAL不需要作为用来恢复数据的备份，即没有任何引用指向这个WAL文件。
    - TTL进程引用：超时时间（默认10min）。
    - 如开启replication，该进程在所有集群备份前引用。

### store ###
- ![191018.store.png](https://img-blog.csdnimg.cn/20191018195616576.png)
- 一个store存放一个**列族**的数据。
- Memstore：一个store中一个memstore，内存存储对象，满了之后刷到HFile。
  - 实现LSM树的组件：尽量保证数据是**顺序存储**到磁盘上，并有频率的**整理**，确保顺序性。从而在频繁的数据变动下保持系统读取的稳定性。
- HFile：MemStore满了之后生成新的HFile，由块组成，每个块包括：
  - Data：数据块。
    - BlockType：数据块等。
    - 多个Cell：KeyValue键值对。
  - Meta：元数据块，文件关闭时写入。
  - FileInfo：文件信息，比如最后一个key，文件关闭时写入。
  - DataIndex：数据块的索引。
  - MetaIndex：元数据块索引。
  - Trailer：各个块的偏移值。

> HDFS上的文件只能创建、删除、追加，不能**修改**。<br>
> WAL按照**写入顺序**排序，经过Memstore按照**rowkey顺序**排序。

## 访问流程 ##
### 增删改实质 ###
- 写入顺序WAL->Memstore->HFile
- 新增Cell,在HDFS上新增一条数据。
- 修改Cell,新增版本号更大（自定义版本号）的一条数据。
- 删除Cell,新增一条Delete类型的数据，即墓碑标记，相当于逻辑删除。

### 写入顺序 ###
1. WAL：基于HDFS，虽然已经持久化，但是时暂存日志，不区分store，不能直接读取。
2. Memstore：整理成LSM树。
   1. 刷写（定时任务）之前，memstore容量如果到达阻塞阈值会暂停memstore的写入，可通过调大JVM堆解决。
3. HFile：memstore达到尺寸上限或者刷写间隔，刷到HDFS
   1. Minor Compaction：store中的多个HFile合并为一个，达到TTL的数据会被移除。
   2. Major Compaction：store中的所有HFile合并为一个，过期数据和手动删除的数据会被移除。

### 查询架构 ###
- ![191022.query.png](https://img-blog.csdnimg.cn/20191022003029349.png)
1. 从zk上查有hbase:meta表的region server。
2. 连接该region server获取所有region的行键范围，并且缓存该meta信息。
3. 直连包含目标rowkey的region server操作。

### 查询顺序 ###
1. BlockCache：缓存block块。一个region server只有一个blockCache
2. Memstore
3. HFile

## API ##
- checkAndPut：CAS的应用，Hbase保证原子性。
- append：value后面增加字节组。
- increment：long类型value+1。
- RowMutations：组合多个操作为原子操作。
