[toc]
## 文件事件模型 ##
- ![191112.redisthread.png](https://img-blog.csdnimg.cn/20191112093321260.png)
- 事件驱动设计，采用[reactor模式](https://www.cnblogs.com/doit8791/p/7461479.html)——每当一个Event输入到Service Handler之后，该Service Handler会主动的根据不同的Event类型将其分发给对应的Request Handler来处理。

### 套接字 ###
- 每个套接字准备好执行连接应答、写入、读取、关闭等操作时，都会产生一个文件事件。
- 一个服务通常会连接多个套接字。
- 文件事件：对套接字操作的抽象。
  - 套接字可读（客户端write,close）、新的可应答套接字（客户端connect），套接字产生AE_READABLE事件。
  - 套接字可写（客户端read），套接字产生AE_WRITABLE事件。
  - 同一个套接字**优先处理AE_READABLE**事件。

### I/O多路复用程序 ###
- 监听多个套接字，并通过**队列**传递产生了事件的套接字给下游事件分派器。<br>![210218.redis.queue.png](https://img-blog.csdnimg.cn/2021021823403922.png)
- [基于 epoll 实现](https://blog.csdn.net/wxy941011/article/details/80274233)。epoll优势：
  - 不限制最大连接数，上限为文件描述符。
  - 只关注活跃连接，不随socket增长线性下降。
  - [mmap共享内存](https://blog.csdn.net/luckywang1103/article/details/50619251)，避免fd列表在内核和用户之间copy。

#### 3种流程 ####
- select：
  - 用户态copy fd数组至内核态。
  - 内核遍历fd，查看是否有IO事件。此时select阻塞。
  - 返回给用户态可读fd个数，用户态遍历具体可读fd。
- poll:同select，仅取消select 1024个fd的限制。
- [epoll](https://mp.weixin.qq.com/s/JHqVY02mMJIpuZ4s9XOrVg)：
  - 内核中保存fd数组，每次变更时epoll_ctl通知内核。
  - 内核不轮询，通过异步IO事件唤醒用户线程。
  - 内核将有IO事件的fd返回给用户态。<br>![210415.epoll.png](https://img-blog.csdnimg.cn/20210416002648243.png)

### 文件事件分派器 ###
- 接收队列中的套接字，并根据**事件类型**调用相应的事件处理器。
- 上一个套接字关联的事件处理器执行完毕，才继续处理下一个套接字。
- redis**单线程**指的是[事件分派器单线程](https://blog.csdn.net/dreamwbt/article/details/81148588)。

### 事件处理器 ###
- 是一个个函数，处理对应事件。 	常用的有：
  - 连接应答处理器：redis服务器初始化时，关联**服务器监听套接字**产生的AE_READABLE事件。事件在客户端connect时产生，创建套接字。
  - 命令请求处理器：客户端连接成功后，关联**客户端套接字**的AE_READABLE事件。该事件在客户端发送请求时产生。
  - 命令回复处理器：服务器有命令回复需要传送给客户端时，关联**客户端套接字**的AE_WRITABLE事件。事件在客户端尝试读取时产生。

## 时间事件模型 ##
- 分为定期事件和周期事件。
- 文件事件和时间事件之间是合作关系，服务器**轮流处理**，不会抢占，所以时间事件实际处理事件通常比设定的晚一点。

## 命令执行流程 ##
- [redis命令执行流程分析](https://blog.csdn.net/houjixin/article/details/27184299)
- ![210218.redis.event.png](https://img-blog.csdnimg.cn/2021021823403929.png)

## 高效的单线程 ##
- 纯内存操作。mysql之类的还涉及读写磁盘。
- 非阻塞I/O多路复用。不用被之前客户端的请求阻塞。
- 单线程没有线程切换开销。
- C语言，效率高。