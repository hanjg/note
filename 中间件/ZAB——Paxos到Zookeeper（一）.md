[toc]
## 简介 ##
- ZAB（Zookeeper Atomic BroadCast）协议。
- 为zk专门设计的支持**崩溃恢复**的**原子广播**协议。
- zk使用zab协议在**主备模式**的架构中保持集群各副本数据的**一致性**。

## 崩溃恢复模式 ##
### 条件 ###
- 服务框架重启。
- Leader异常：网络中断、崩溃退出、重启。

### 目的 ###
- 选取新Leader。
- Follower和Leader数据同步。
  - 已经被Leader提交的Proposal能够被所有Follower提交。
  - 跳过已经被丢弃事务的Proposal。

### 步骤 ###
> ZXID：事务ID，共64位，32位epoch标识leader周期，32位标识事务编号
1. [Leader选举](https://blog.csdn.net/qq_40369829/article/details/106935410)。Leader选举为提出Proposal中最大ZXID的机器，对epoch+1。
2. Leader以Proposal紧接着Commit消息的形式同步数据给Follower。
3. 包含上一个leader周期未提交Proposal的Follower启动时，会被Leader要求回退到过半机器提交的最新Proposal。

## 消息广播模式 ##
### 条件 ###
- 过半Follower完成和Leader的状态同步。

### 目的 ###
- 主备数据一致性。
  - [两阶段提交](https://blog.csdn.net/qq_40369829/article/details/104012837)移除回滚逻辑。
  - 基于FIFO的TCP，保证消息接收和发送的顺序性。

### 步骤 ###
- ![200618.zkbroadcast.png](https://img-blog.csdnimg.cn/20200624000752352.png)
1. Leader为每个事务请求生成Proposal，分配ZXID。 
2. Leader广播Proposal。
3. Follower事务日志落盘，返回Ack。
4. Leader收到超过半数Follower的Ack后，广播Commit消息通知提交。
5. Leader自身提交。
6. Follower收到Commit消息之后提交。

## 和Paxos的关系 ##
- Zab是multi-Paxos的变体。
  - [Paxos共识算法详解](https://juejin.im/post/5cb00706e51d456e3428c0c9#heading-10)。
  - [basic-paxos和multi-paxos](https://zhuanlan.zhihu.com/p/25664121)。

### 相同 ###
- Leader协调多个Follower。
- Leader等待超过半数Follower反馈后提交。
- 每个Proposal包含一个epoch值，标识当前的Leader周期。

### 不同 ###
- 目的不同。
  - ZAB：构建高可用分布式主备系统。
  - Paxos：构建分布式一致性状态机系统。