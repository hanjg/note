[toc]
## 消息队列 ##
- 作用：
  - **解耦**。解耦出调用方，特别是被调用方较多，且经常变更时。
  - **异步**。减少rt。
  - **削峰**。类似缓存，抗峰值流量。
- 缺点：
  - **可用性**降低。MQ挂掉影响这个系统。
  - **复杂度**提高。消息重复、丢失、顺序等问题。
  - **一致性**。部分消费成功，部分不成功情况。

### rabbitMQ ###
- 使用AMQP模型。punlisher发布消息到exchange，根据routes发送到绑定的queue，push或者pull给consumer。<br>![](https://www.rabbitmq.com/img/tutorials/intro/hello-world-example-routing.png)
- 其他概念：
  - broker:消息队列服务器。
  - vhost：broker里面设置，用于用户分离。
  - channal：每个客户端连接可建立多个channal，代表会话。
- 交换机有四种类型。
  - 直连交换机：单播，根据routing key投递给队列。
  - 扇形交换机：广播，发送到绑定的所有队列，无视routing key。
  - 主题交换机：多播，根据routing key和队列绑定规则发送。
  - 头交换机。 
- [官方文档](http://rabbitmq.mr-ping.com/AMQP/AMQP_0-9-1_Model_Explained.html)，[入门教程](https://www.jianshu.com/p/dae5bbed39b1)。

### kafka ###
- 多订阅者的Topic，每个topic分为多个partition，并行消费。通过offset来标识消费者已经消费的消息。<br>![](http://kafka.apachecn.org/10/images/log_anatomy.png)
- 一个consumer group（逻辑消费者）由多个consumer组成，消费一个topic。<br>![](http://kafka.apachecn.org/10/images/consumer-groups.png)
- [官方文档](http://kafka.apachecn.org/intro.html)

## 对比 ##
- rabbitMQ：erlang语言开发，依靠活跃的开源社区支持。
  - 数据一致性、稳定性和可靠性高。 
- rocketMQ：阿里出品。
- kafka：业界标准.
  - 可扩展，高性能，多partition可容错。适合大数据领域等，可改造可靠性。
- [参考](https://segmentfault.com/a/1190000016376216)。