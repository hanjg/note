[toc]
## 原理 ##
### 架构 ###
- ![200122.dubboarch.png](https://img-blog.csdnimg.cn/20200122151424995.png)
0. 服务容器启动，加载运行提供者。
1. 提供者启动时，向注册中心**注册**服务。
	1. spring refresh之后，通过配置组装URL，[javassist](https://zhuanlan.zhihu.com/p/149196141)动态代理(快)创建provider实现类。
	2. 默认dubbo协议暴露服务，注册provider。
2. 消费者启动时，向注册中心**订阅**服务。
	1. 通过配置组装URL，注册consumer信息。
	2. 拉取Provider的配置、路由等。
	3. 创建代理类进行远程通信。
3. 注册中心返回提供者列表给消费者，变更时基于长连接**推送**。
	1. zk节点数据结构：<br>![210512.dubbo.zk.png](https://img-blog.csdnimg.cn/20210513001713695.png)
4. 消费者基于负载均衡选一个提供者调用。
5. 提供者和消费者定时上报监控中心。

### 调用过程 ###
- 底层基于[netty](https://juejin.cn/post/6844903712435994631)，服务端主从reactor处理请求。<br>![200122.dubbocall.png](https://img-blog.csdnimg.cn/20200122151831149.png)
1. consumer调用代理类，通过路由找到provider列表，lb其中一个远程调用。
2. 记录请求和请求id等待响应。
3. provider根据参数找到代理类，执行方法，返回值带上请求id。
4. consumer收到响应后根据id找请求，将响应塞到请求对应的future对象里，唤醒等待的线程。

## 特点 ##
- 优点：
  - 透明化的远程方法调用。只需简单配置即可像本地方法一样使用。
  - 软负载均衡和容错。请求失败后切换到其他提供者重试。
  - 服务自动注册。启动时自动注册在注册中心上。
  - 服务监控和治理。
- 缺点：
  - 只支持java。

## SPI ##
### 流程 ###
- [获取扩展类](https://www.cnblogs.com/LUA123/p/12460869.html)。
	- 读取约定配置在：META-INF/dubbo/internal/ 下。
	- 读取并解析每一个文件，按等号分割为 name-class键值对。
	- 根据类名生成Class对象，根据Class对象的的特点进行相关的缓存以及name到Class对象的缓存。
- IOC注入依赖。反射注入依赖。

### 特点 ###
- dubbo SPI按需加载，节省资源。

## 注意点 ##
### 异常处理 ###
- [server](http://xurui.pro/2018/05/29/%E5%9F%BA%E4%BA%8Edubbo%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%BA%94%E7%94%A8%E4%B8%AD%E7%9A%84%E7%BB%9F%E4%B8%80%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86/)：抛出的异常需要在jar包定义，否则通过ExceptionFilter时会被封装成 **RuntimeException** ，client无法catch指定异常。
- client：hessian在JavaDeserializer解码异常时用cost最小构造器实例化对象，异常最好加上无参构造器，否则如果使用非基础类型的构造器实例化对象有可能NPE。

### 发布调用异常 ###
- 集中在新实例。极有可能provider端的accept队列满了导致。
- 集中在老实例。
  - 没有[优雅关机](https://jaskey.github.io/blog/2019/09/30/spring-boot-dubbo-graceful-shutdown/)。
  - 大量consumer断连，但是仍然后部分流量到provider。断连事件和业务请求争抢有限的Netty IO资源，业务请求被延迟。解决方案：
    - provider关机时先发送readonly，等待一段时间之后再摘除zk节点。
    - consumer收到provider下线请求后，将代理类设为不可用，等待后再销毁断开连接。

## 文档 ##
- [官网](https://dubbo.apache.org/zh-cn/index.html)。
- [dubbo和spring cloud](https://zhuanlan.zhihu.com/p/85012048)。
- [dubbo参考](https://aobing.blog.csdn.net/article/details/109018922)