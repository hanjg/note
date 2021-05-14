[toc]
## 主备 ##
- binlog实现。

### 主备同步 ###
#### 双M结构 ####
- ![201117.ms1.png](https://static001.geekbang.org/resource/image/20/56/20ad4e163115198dc6cf372d5116c956.png)
- 备库设成readonly（对super用户无效，比如同步更新的线程）：
  - 取数等放在备库，可以防止误操作。
  - 防止切换过程的bug，造成双写主备不一致。
  - readonly属性可以用来判断节点角色。
- server id解决循环复制问题。
  - binlog中记录业务操作的mysql的sever id。
  - 每个库收到binlog后，丢弃和自己server id相同的binlog。

#### 同步流程 ####
- ![201117.ms2.png](https://static001.geekbang.org/resource/image/a6/a3/a66c154c1bc51e071dd2cc8c1d6ca6a3.png)
- 流程：
  - 备库使用change master，设置主库的参数和开始请求binlog的位置。
  - 备库执行start slave， 启动io_thread建立和主库连接，启动sql_thread读取日志并执行。
  - 主库校验用户，从请求的位置开始读本地**binlog**，发送给备库。
  - 备库接收binlog之后写本地中转日志**relay log**。
  - sql_thread解析中转日志并执行。
- 默认异步复制，如果需要保证不丢数据，可使用[半同步](https://www.cnblogs.com/kevingrace/p/10228694.html)。
  - master commit之后等待一个备库ack，再返回客户端。

#### 并行复制 ####
- 备库并行回放binlog。
- [原则](https://time.geekbang.org/column/article/77083)：
  - **同一行的两个事务**分发到一个worker中。防止更新覆盖。
  - **同一个事务**分发到一个worker中。防止破坏事务隔离性。
- mysql 5.6支持按db维度分发。对一个实例一个db的场景无效。
- MariaDB利用**组提交中的事务可并行**的特点，备库**模拟主库**并发度。
  - 能在一组提交的事务(处于commit)，一定不会修改同一行，将相同的commit_id计入binlog。
  - 主库可并行执行，备库也可并行执行。相同commit_id的事务可以分发到不同的worker。
- mysql 5.7 优化mariadb的并发度，同时处于**redolog prepare阶段和处于commit阶段**的事务备库可以并行。
  - binlog组提交参数增加处于Prepare阶段的事务，提高并发度。

### 主备延迟 ###
#### 定义 ####
- 主备同步关键时刻：
  - 主库执行完事务，写binlog，T1.
  - 备库接收到binlog，T2.
  - 备库执行完事务，T3.
- 主备延迟：主备执行完事务的时间差，即T3-T1.

#### 来源 ####
- 备库配置低于主库。
- 备库压力大，承担过多的查询任务，消耗过多cpu。
- 大事务。比如一次性delete过多，大表DDL。
- 备库并行复制能力不足。

### 主备切换 ###
- 可靠性优先：<br>![201117.realiable.png](https://static001.geekbang.org/resource/image/54/4a/54f4c7c31e6f0f807c2ab77f78c8844a.png)
  - 备库seconds_behind_master较小时，主库改成只读。
  - 等待备库 seconds_behind_master = 0，备库改成可读写。
  - 请求切到备库。
- 可用性优先。
  - 不等待主从延迟归0，直接切写。可能造成[数据不一致或主键冲突](https://time.geekbang.org/column/article/76795)。

## 主从 ##
### 主备切换逻辑 ###
- ![201119.mbs.png](https://static001.geekbang.org/resource/image/00/53/0014f97423bd75235a9187f492fb2453.png)
- [GTID](https://time.geekbang.org/column/article/77427)=server_uuid:gno 。
  - gno初始值为1，提交事务时分配给事务并自增，事务回滚不分配。
  - 每个MySQL实例都维护了一个 GTID 集合标识：**这个实例执行过的所有事务**。
- 新备库在主备切换时使用GTID寻找的和从库的同步位点，从库执行start slave的逻辑：
  - 从库建立和新主库的连接。
  - 从库把自身GTID集合发给新主库。
  - 新主库**计算比从库多的GTID**。
    - 如果这些GTID对应的本地binlog已经删除，异常。
    - 如果全部包含，从第一个多出来的binlog开始顺序发给从库。
- 指定GTID执行空事务，可以不执行事务但是保存对应GTID的binlog。

### 过期读处理方案 ###
- **强制走主库**。放弃扩展性。
- **sleep**。延迟时间不确定，仍然可能过期读。
- **判断主备无延迟**。根据从库GTID等，判断收到的binlog都执行，但是可能有些binlog还未传输，仍然过期读。
- **半同步+确认无延迟**。一个从库ack binlog之后返回，多从场景仍然可能过期读。
- **等待位点或GTID**。等待从库执行到指定的位点再查询，超时则查主库。
- [详细](https://time.geekbang.org/column/article/77636)

## 可用性检测 ##
### 外部检测 ###
- select 1。连通性检查。
- mysql> select * from mysql.health_check。检查并发线程数过多的情况。
- mysql> update mysql.health_check set t_modified=now()。检查磁盘爆满的情况。
> 随机性、滞后性性。

### 内部检测 ###
- performance_schema库file_summary_by_event_name表，统计语句执行时间。
- [其他检查方法](https://time.geekbang.org/column/article/78134)。

## 相关 ##
- [mysql（一）——架构和执行流程](https://blog.csdn.net/qq_40369829/article/details/100154362)
- [mysql（二）——索引](https://blog.csdn.net/qq_40369829/article/details/100154514)
- [mysql（三）——日志](https://blog.csdn.net/qq_40369829/article/details/100154560)
- [mysql（四）——快照读](https://blog.csdn.net/qq_40369829/article/details/91359489)
- [mysql（五）——锁](https://blog.csdn.net/qq_40369829/article/details/100154535)
- [mysql（六）——高可用](https://blog.csdn.net/qq_40369829/article/details/110413780)
- [mysql（七）——部分语句实现](https://blog.csdn.net/qq_40369829/article/details/110413795)