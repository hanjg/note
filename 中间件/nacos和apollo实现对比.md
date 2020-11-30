[toc]
## nacos ##
### 架构 ###
- ![](https://user-gold-cdn.xitu.io/2020/1/19/16fbbc2c2bd9a514)
- client -> server <- admin。
- 服务端持久化可简化为内置的**Derby**内存数据库。

### 配置更新 ###
- ![](https://upload-images.jianshu.io/upload_images/5417792-bc24eecd1b358e28.jpg)
- HTTP从server**长轮询**拉配置。请求**延时阻塞**在服务端队列中。
- 服务端task扫到key的更新，变更后的配置写入响应对象，返回结果。
- 没有变更阻塞29.5s后(客户端超时30s)检查配置，检查结果写入响应对象，返回结果。
- 客户端检查配置的**md5**，不一致调用listener通知变更事件。

### 概念 ###
- nacos配置层级和项目对应关系：<br>![](https://upload-images.jianshu.io/upload_images/18901093-7ca525eb69f1af23.png)

## apollo ##
### 架构 ###
- 最简部署架构。<br>![](https://mmbiz.qpic.cn/mmbiz_png/ELH62gpbFmGdnIjxDT7AOQyZgl2KQnz6SNgVAvt0zKibxC0IqAQxvjkMibc0k8ibk1fZ0d7UGLSf96ibupPJ2jueOg/640)
- [分布式架构](https://mp.weixin.qq.com/s/-hUaQPzfsl9Lm3IqQW3VDQ)。

### 配置更新 ###
- 服务端定期扫更新配置<br>![200915.apollo.server.png](https://img-blog.csdnimg.cn/20200915000245899.png)
  - releaseMessage实现方式：
    1. admin server写mysql表。
    2. config server的task扫更新记录。
- 客户端缓冲配置（内存+文件）。<br>![200915.apollo.client.png](https://img-blog.csdnimg.cn/20200915001356738.png)
  - 推送实现方式：同nacos **http长轮询**。
 
## 对比 ##

||apollo|nacos|
|--|--|--|
|推送方式|HTTP长轮询|HTTP长轮询|
|客户端存储|内存+文件|内存+文件|
|最小部署|config+admin+portal+mysql|sever+内嵌db或mysql|
|权限管理|复杂|简单|

## 参考 ##
- [nacos文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)。
- [nacos更新原理](https://www.jianshu.com/p/acb9b1093a54)
- [nacos实现原理](https://juejin.im/post/6844904050840993805)。
- [apollo文档](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1)。