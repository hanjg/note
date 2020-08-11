[toc]
## 高可用 ##
- 主从复制+哨兵模式。

## 复制 ##
- 命令：``` SLAVEOF ip```<br>![200805.rediscopy.png](https://img-blog.csdnimg.cn/202008070005559.png)

### 完整重同步 ###
- 步骤
	1. 从向主发送```SYNC```
	2. 主执行BGSAVE，生成**RDB文件**，并使用**缓冲**记录快照之后的所有写命令。
	3. 主发送RDB文件至从，从阻塞请求加载RDB文件。
	4. 主发送缓冲区命令至从，从执行到同步。
- 缺点：消耗资源，主持久化、网络传输RDB、从阻塞请求。

### 部分重同步 ###
- 实现方式：
  - 主从复制偏移量**offset**。字节数的位移。
  - 主服务器的**复制积压缓冲区**。
    - offset滞后于缓冲区，需要完整重同步。
    - offset在缓冲区中，部分重同步。
    - offset和缓冲区一致，不重同步。
  - 服务器**运行ID**。判断从节点身份，和缓存区、offset一起决定是完整还是部分重同步。

### 命令传播 ###
- 重同步完成之后：发送写命令+写复制积压缓冲区。<br>![200805.redispsync.png](https://img-blog.csdnimg.cn/20200807000555802.png)

## 哨兵模式 ##
- Sentinel模式（特殊模式下的redis实例）监视多个主从服务器，主服务器下线时，将从服务器升级为新的主服务器。

### 正常运行 ###
- ![200807.run.png](https://img-blog.csdnimg.cn/20200807001533628.png)
- Sentinel实例之间只需要创建命令连接。

#### 命令连接 ####
- 连接主从服务器：发送**INFO命令**，默认10s一次，获取服务器当前信息（主服务还包括有哪些从节点）。
- 连接其他Sentinel实例：下线检测、故障。

#### 订阅连接 ####
- 连接主从服务器：
  - 向服务器**_sentinel_:hello频道**发送自身Sentinel实例和监视的主服务器信息，默认2s一次。
  - 订阅服务器_sentinel_:hello频道，更新Sentinel和主从服务器拓扑，包括发现未知的Sentiel实例。

### 检查下线 ###
- ![200807.down.png](https://img-blog.csdnimg.cn/20200807001533622.png)

#### 主观下线 ####
- Sentinel默认每秒**ping**一次（主从服务器，其他Sentinel），超过阈值无响应。

#### 客观下线 ####
- 认为主观下线时，向其他Sentinel发送命令询问节点是否下线。
- 如果多数Sentinel回复节点下线，则认为客观下线。

### 故障转移 ###
- ![200807.migrate.png](https://img-blog.csdnimg.cn/20200807001533687.png)

#### Leader选举 ####
- 在监视这个master的Sentinel中选举Leader节点操作故障转移。[raft算法](https://blog.csdn.net/DONGWEIJHZHANGLI/article/details/92407376)。
- 步骤：
  1. 做主观下线的Sentinel节点像其他Sentinel节点发送命令，要求将自己设置为Leader。
  2. 接收到的sentinel同意最先收到的Sentinel节点命令（**先到先得**），选该节点为Leader，发送回复。
  3. 如果做主观下线的Sentinel节点发现自己的票数已经**超过半数**，则成为Leader。
  4. 如果一定时间内未选出Leader，那么将等待一段时重新进行选举。

#### 故障恢复 ####
- 从服务器中升级一个至主服务器。
- 其他从服务器复制这个新的主服务器。
- 下线的主服务降级为从服务器,上线后，复制新的主服务器。
- ![200807.new.png](https://img-blog.csdnimg.cn/202008070015336.png)