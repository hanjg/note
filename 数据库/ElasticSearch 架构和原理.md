[toc]
## 部署架构 ##
- ![200824.esphysic.png](https://img-blog.csdnimg.cn/20200826000209675.png)
- **master节点**。
  - 管理元数据，负责索引创建、删除、数据rebalance等。
  - 通常奇数个master节点。
- **数据节点**。
  - 存储数据和倒排索引，负载数据的索引和检索，负载较高。
- **协调节点**。
  - 处理客户端请求，转发至相关的数据节点，并聚合结果返回客户端。

## 存储架构 ##
- ![200824.eslogic.png](https://img-blog.csdnimg.cn/20200826000209955.png)
- 索引。index。
  - 文档的集合，由多个分片组成。
- 分片。shard。
  - 一个索引的数据保存在多个分片中。
  - 索引的分片数在创建索引的时候确定，无法更改，数据节点扩容时分片再平衡。
  - 一个主分片可以有n个副本分片，不在同一个数据节点，冗余备份、分担读请求。
- 分段。segment。
  - 实际存储文档的文件，一个分片多个分段。
- 文档。docoment。
  - 索引和搜索的原子单位。包含多个域。
- 域。field。
  - text会分词，如不需要分词可使用keyword。
- 最小搜索单元。term。
- 倒排索引。
  - 列举所有文档中出现的terms以及它们出现的文档ID和出现频率。
  - 搜索时同样会对关键词进行同样的分词分析，然后查表得到结果。

## 写入过程 ##
- 分片路由规则：默认文档ID hash，也可以指定字段。

### 请求链路 ###
- 协调节点转发至主分片数据节点。
- 主分片写入。
- 主分片写入成功后，并发写入所有副本分片，所有副本写入完成后返回客户端。

### 创建文档 ###
#### write ####
- document写入**In-memory buffer**和**内存translog**。<br>![210224.es.write.png](https://img-blog.csdnimg.cn/202102241309124.png)

#### refresh ####
- 默认1s一次refresh **In-memory buffer**，写入**文件系统缓存**，构成新的segment。
- 此时文档可以**搜索到**。<br>![210224.es.refresh.png](https://img-blog.csdnimg.cn/2021022413100859.png)

#### fsync ####
- 默认5s一次，刷**translog**到磁盘。translog写到磁盘之前可能丢失数据。
  - 为了不丢数据，可配置为每次写内存translog后fsync到磁盘，但是会降低吞吐。
- 默认30min一次或traslog文件过大时，fsync 文件系统缓存的 **segment** 到磁盘，清除translog。<br>![210224.es.fsync.png](https://img-blog.csdnimg.cn/20210224131218400.png)

#### merge ####
- 每1s生成一个segment文件，数量较多，问题如下：
  - 跟多的元数据管理segement。
  - 删除操作导致大量无效数据。
  - 查询遍历每个segment，性能低下。
- 单独的归并线程merge segment，默认最大segement 5G。

### 删除文档 ###
> 追加式存储无法直接删除。
- 一个segment维护一个del文件，删除文档时在**del文件里标记**。
- 返回查询结果时过滤。
- segment merge时标记删除的文档才被删除。

### 更新文档 ###
- 查找原文档得到原文档版本号。
- 修改后的文档，用原版本号+1走**创建新文档**流程。
- del文件里**标记删除旧版本文档**。

## 读取过程 ##
### 查询文档 ###
- 协调节点根据文档ID路由至数据节点（分片可能主分片也可能副本分片）。
- 数据节点返回协调节点文档。
- 协调节点返回客户端结果。

### 搜索文档 ###
- 协调节点广播查询条件至相关的数据节点（分片可能主分片也可能副本分片）。
- 数据节点返回给协调节点结果集文档ID（轻量）。
- 协调节点汇总结果文档ID。
- 协调节点根据文档ID查询文档，返回客户端结果

## 参考 ##
- [ElasticSearch底层原理浅析](https://blog.csdn.net/zkyfcx/article/details/79998197?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)
- [Elasticsearch架构原理](https://www.jianshu.com/p/5b88e95a9e80)