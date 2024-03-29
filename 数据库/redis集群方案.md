[toc]
## redis cluster ##
### 架构 ###
- redis集群两两之间互相通信，并将16384个哈希槽分配给每个节点。
- 当需要在 Redis 集群中set一个 key-value 时，redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽。
- redis 会根据节点数量大致均等的将哈希槽映射到不同的节点。<br>![](https://img-blog.csdn.net/20180404221833788?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMzY5ODI5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 特点 ###
#### 优点 ####
- 部署简单。redis自身支持，无需其他节点。
- 支持高可用。

#### 缺点 ####
- 客户端需要定制。需实现redis集群协议。

### 槽迁移 ###
- 槽迁移整体流程。<br>![201202.redisrebalance.png](https://img-blog.csdnimg.cn/20201229191731254.png)
- 迁移key的过程。<br>![201202.migaratekey.png](https://img-blog.csdnimg.cn/20201229191851982.png)
- 槽迁移过程中，如果请求源节点，而key被迁移到目标节点会返回ASK错误，客户端这**一次**请求转向目标节点。<br>![201202.ask.png](https://img-blog.csdnimg.cn/20201229191850954.png)
- 槽迁移结束，如果请求源节点发现对应的槽已迁移，返回MOVED错误，客户端**之后**这个槽的请求都转向目标节点。

### 容错机制 ###
- [集群模式](https://blog.csdn.net/qq_40369829/article/details/107948395)。

## codis ##
### 架构 ###
- 基于redis 2.8.13开发,在redis和client中间插入Proxy层，内存维护槽位（key分组）和Redis实例映射关系，zk持久化映射关系。<b>![200123.codisarch.png](https://img-blog.csdnimg.cn/20200123161018149.png)

### 特点 ###
#### 优点 ####
- 分布式逻辑和存储逻辑分离。客户端无需关注服务端部署，client和server可各自升级，proxy扩容方便。
- 支持pinpline等redis cluster不支持的命令。
- 大规模生产使用过，可靠。redis cluster待考验。

#### 缺点 ####
- 部署复杂。服务节点较多，除了redis还需proxy和zk。
- 性能略微受损。proxy-server低于1ms的rt

### 参考 ###
- [Codis作者黄东旭细说分布式Redis架构设计和踩过的那些坑们](https://www.open-open.com/lib/view/open1436360508098.html)
- [为什么大厂都喜欢用 Codis 来管理分布式集群](https://juejin.im/post/5c132b076fb9a04a08218eef)
- [Codis与Redis Cluster集群方案对比](https://blog.csdn.net/tiandao321/article/details/88353128)

## 客户端分片 ##
### 架构 ###
- redis节点之间互不通信，client直连多台redis（主从），在客户端做hash。

### 特点 ###
#### 优点 ####
- 架构简单，可靠。
- 支持原生的redis命令。

#### 缺点 ####
- 分布式逻辑和存储逻辑耦合在客户端。

### 参考 ###
- [Jedis客户端分片的实现](https://www.jianshu.com/p/af0ea8d61dda)