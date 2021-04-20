[toc]
## 架构 ##
- ![200705.kafka.struct.png](https://img-blog.csdnimg.cn/20200708134216538.png)

## 元数据 ##
- broker集群成员列表。维护在zk的 ```/brokers/ids```下的**临时节点**。
- 控制器。
  - broker之一，通过成功创建```/controller```节点判定。
  - 监听各broker zk节点状态，维护分区首领和追随者列表，并通知各个broker。

## 复制 ##
- 每个分区多个副本。
### 首领副本 ###
- 所有生产者和消费者请求经过首领副本，追求一致性。

### 追随者副本 ###
- 从首领复制消息，保持一致。
  - 主动从首领**pull**消息。
  - 未满足条件降级为**不同步副本**。
    - 与zk之间有活跃会话，默认6s超时。
    - 默认10s没有从首领pull。
    - 从首领获取消息延时默认10s之内。
- 首领崩溃时其中一个升级为首领。

> 副本在同步和非同步之间快速切换，极有可能是GC导致。

## 处理请求 ##
### 元数据请求 ###
- 元数据请求时机：
  - 客户端向任意broker定时刷新。
  - 客户端收到非首领错误，Pull 任意broker。<br>![200702.metarequest.png](https://img-blog.csdnimg.cn/20200708134216616.png)

### 生产请求 ###
- 生产请求。生产者写broker。
- [acks参数控制生产者认为写入成功的条件](https://blog.csdn.net/qq_40369829/article/details/107203986)。

### 获取请求 ###
- 获取请求。消费者、追随者副本读broker。
- broker根据客户端请求的数量上限下限、最大等待时间，选择时机将消息从文件系统缓存**零复制**发送到客户端。
- **所有同步副本**复制消息之后，才允许消费者读取。

## 分区存储 ##
- 分区分为n个片段（包括1个正在写入的活跃片段）。
- 片段分为两部分：
  - 干净部分：清理过，每个键只有一个值。
  - 污浊部分：清理之后写入。
- 清理：相同key保留最新的值。

## 快的原因 ##
- 写入broker
  - [顺序写磁盘](https://mp.weixin.qq.com/s/5Mcs1zmoyB6b03c1NGotZA)。
  - [mmap](https://mp.weixin.qq.com/s/-W3oh_fFugZLMF8P5RFSjQ)（Memory Mapped Files）。
- 读取broker
  - sendfile
  - [批量压缩消息](https://mp.weixin.qq.com/s/5Mcs1zmoyB6b03c1NGotZA)，节省网络资源。

### read+write ###
- 4次上下文切换、2次CPU复制、2次DMA复制。<br>![210323.kafka.read.write.png](https://img-blog.csdnimg.cn/20210323002526898.png)

### mmap ###
- 用户缓冲区映射到内核缓冲区，避免复制到用户空间。（[page cache](https://blog.csdn.net/qq_34827674/article/details/108756999))。
- 4次上下文切换、内核空间内1次cpu复制、2次DMA复制。<br>![210323.kafka.mmap.png](https://img-blog.csdnimg.cn/20210323002526921.png)

### sendfile ###
- 内核缓冲和socket缓冲复制，减少上限文切换。<br>![210323.kafka.sendfile.png](https://img-blog.csdnimg.cn/20210323002526844.png)
- 2次上下文切换、内核空间内1次cpu复制、2次DMA复制。

### sendfile+DMA Scatter/Gather ###
- 文件描述符复制到socket缓冲，DMA 从内核缓冲区复制网卡。需要硬件支持。<br>![210323.kafka.sendfile.gather.png](https://img-blog.csdnimg.cn/20210421004128315.png)
- 2次上下文切换、2次DMA复制。
