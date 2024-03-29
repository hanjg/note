[toc]
## 数据系统 ##
### 可靠性 ###
- 定义：
  - 执行用户期望的功能。
  - 容忍用户不正当的使用方法。
  - 性能可以符合预期的用例、负载、数据量。
  - 防止未经授权的访问和滥用。
- **硬件故障**。
  - 硬件冗余。
- **软件故障**。节点软件关联，影响范围广。
  - 监控，进程隔离等。
- **人为失误**。
  - 减少操作、测试环境演练、应急预案。

### 可扩展性 ###
- 定义：
  - 描述应对系统负载增长的能力。
- 描述负载。QPS等。
  - twitter粉丝数量剧增，读时查询->写时推送。
- 描述性能。rt（实时，avg+999）、吞吐量（离线）等。
- **应对负载的方式**。
  - 水平扩展。无状态服务水平扩展容易。
  - 垂直扩展。

### 可维护性 ###
- **可运维性**。监控等。
- **简单性**。抽象等简化复杂度，易于理解。
- **可演化性**。易于改变，方便扩展功能。

## 数据模型和查询语言 ##
### 数据模型 ###
- **文档模型**。适合一个文档和其他文档关联很少的场景。
  - 较好的模式灵活性。不强制执行模式，数据可同时包含新旧模式，增加字段较方便。
  - 较好的局部性。适合同时访问文档大部分内容的场景。
- **关系模型**。适合联结，描述一对多、多对一、多对多关系。
  - 数据规范化。
    - 各种id，对人无意义，所以不需要变更。
- **图模型**。适合所有用例都可能产生多对多关系的场景。参考：[图数据库入门](http://neo4j.com.cn/topic/5d396aa3b2f2eb3c4208cc71)
  - 属性图。顶点包含出入边，联结范围可预知。
    - Ciper。
  - 三元组。
    - SPARQL。

### 查询语言 ###
- **声明式**。分层思想，使用方便，隐藏实现细节，可无感知升级。
  - js选择器
- **命令式**。
  - js DOM

## 数据存储和检索 ##
### 数据结构 ###
- 实现的问题和应对方案。

|  |  追加 | 更新 |
|--|--|--|
| 删除记录 | 墓碑记录，合并时丢弃记录 | 直接删除 |
| 崩溃恢复 | WAL | REDO |
| 部分写入 | 校验码发现损坏数据 | double write |
| 并发控制 | 单个写线程顺序追加文件，多个线程读取 | 锁+MVCC|

#### LSM树 ####
- **日志追加流派**。
- 对比B树优点：
  - 写入吞吐大。
    - 写放大低。无需额外的I/O处理部分更新。
    - 一批数据顺序写入SSTable文件，不需要写入树的多个页。
  - 存储碎片少。定期合并回收碎片，B树页不一定能用完。

##### CSV+内存hash表 #####
- CSV追加操作，定期合并，hash表指向key在CSV的offset。
- 优点：
  - 追加和合并都是顺序写，快于随机写。
  - 追加式崩溃时无需担心部分写失效。
- 缺点：
  - hash适合内存查找，量大之后磁盘查找效率低。
  - hash区间查找效率不高。
- 适合场景：key不多，更新频繁。

##### LSM-Tree #####
- SSTable(排序的k-v对)+内存稀疏索引。
- 相对csv+hash优点：
  - 合并高效。文件大于内存空间时可用外部排序。
  - 内存占用少。稀疏索引指向文件分片。
  - 区间查找效率高。同文件分片一次I/O。
- 查询不存在的数据，需要遍历相关的历史文件，可用布隆过滤器优化。
> LSM存储引擎：基于合并和压缩排序文件原理的存储引擎

#### B树 ####
- **原地更新流派**。
- 对比LSM树优点：
  - 读效率高。一个key对应一个索引位置，LSM树需要访问多个段相同key的记录。
  - 支持事务。key单副本通过加锁等可支持。
  - 读写干扰小。没有压缩这类耗资源的干扰操作。
- 优化：
  - 临近叶子节点指针相连，方便顺序查找。B+树。

### 存储引擎 ###
|  |  OLTP | OLAP |
|--|--|--|
| 读写 | 随机读写 | 批处理或流导入，批量读取 |
| 场景 | BC端应用 | 内部分析 |
| 表征数据 | 最新数据 | 历史所有数据 |
| 瓶颈 | rt，磁盘寻道时间 | 吞吐量，磁盘带宽 |

#### 分析模式 ####
- 星型模式：事实表外键关联维度表。

### 列式存储 ###
- 分析效率高。对于宽的事实表，每个列存在一起，按需查找需要的列。
- 适合压缩。
  - 值重复度高的可以用位图编码。 

## 数据编码和演化 ##
### 数据编码格式 ###
- 编码需要保持双向兼容性。这样可以允许系统独立**滚动升级**。
  - **向后兼容**。新代码读取老代码编写的数据。
  - **向前兼容**。老代码读取新代码编写的数据。

#### 语言特定格式 ####
- java：jdk序列化，[kryo](https://blog.csdn.net/varyall/article/details/80215601)等。
- 优点：使用方便。
- 缺点：
  - 集成困难。限制于使用的语言。
  - 安全问题。解码过程需要实例化任意的类，即可执行任意代码。
  - 兼容问题。为了快速简单编码数据，忽略向前向后兼容。
    - 如jdk序列化不指定serialVersionUID则无法向后兼容。
    - kryo不支持向后兼容。
  - 效率低。jdk编码慢、占空间。

#### 文本格式 ####
- Json,xml,csv等。
- 优点：可读性强。
- 缺点：
  - 数字编码模糊。
    - xml和csv不区分数字和字符串。
    - Json区分数字和字符，但是不区分整数和浮点数，不指定精度，2^53以上的长整数(十进制16位)会[失真](https://segmentfault.com/a/1190000017545048)。
  - 不支持二进制字符串。需要Base64编码，增加1/3空间。

#### 二进制格式 ####
- 基于模式，protocal Buffers等。
- 优点：
  - 编码紧凑。
  - 可根据模式校验前后兼容性。
    - protocal Buffers根据**字段标签**和**数据类型**校验模式的演化。
- 缺点：
  - 无可读性。

### 数据流模式 ###
#### 基于数据库 ####
- 对于旧模式数据。
  1. 重写或迁移成新模式。mysql alter table，赋予列默认值。
  2. 前后兼容。如对于hbase新增列
    - 旧模式代码忽略新列。
    - 新模式代码默认旧模式中的新列为null。
> 注意旧模式代码更新新模式数据，有可能丢失新模式中的**新列**。
- 归档数据不可变，可考虑紧凑的无演化编码（avro）或分析友好的列存储。

#### 基于服务 ####
- 网络服务。
  - REST理念。基于HTTP原则，URL标识资源。
- RPC将远程请求模拟成本地方法。
  - 由于网络原因，执行时间长、请求结果不可预测，如果重试需要保证幂等性。
  - 大对象序列化可能有问题。
  - c/s如果跨语言存在兼容问题。如js的长整数。

#### 基于消息传递 ####
- 消息代理。[如Kafka](https://blog.csdn.net/qq_40369829/article/details/107204190)
  - 异步。提高可用性、减少rt，削峰。
  - 发布。一对多发布。
- 分布式 [Actor框架](https://www.jianshu.com/p/449850aa8e82) 。如Akka, Erlang OTP。　
  - 集成消息代理和单进程中的Actor编程模型。