[toc]
## 安装 ##
### java ###
- zk和kafka的运行环境。
- [linux下的jdk安装](https://blog.csdn.net/qq_40369829/article/details/79900135)

### zookeeper ###
- 下载zk：[https://zookeeper.apache.org/releases.html](https://zookeeper.apache.org/releases.html)
- 解压。```tar -zxvf apache-zookeeper-3.6.1-bin.tar.gz```。
- 创建数据文件夹、配置路径下创建zoo.cfg。
```sh
cd /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/conf
cp zoo_sample.cfg zoo.cfg 
```
```txt
tickTime=2000
dataDir=/usr/local/zookeeper/apache-zookeeper-3.6.1-bin/dataDir
dataLogDir=/usr/local/zookeeper/apache-zookeeper-3.6.1-bin/dataLogDir
clientPort=2181

initLimit=10
syncLimit=5
admin.serverPort=4888
server.1=localhost:2888:3888
```

- bin目录下启动zk。
```sh
./zkServer.sh start
./zkServer.sh status
``` 

- 客户端操作。
```sh
./zkCli.sh -server 127.0.0.1:2181
help
```

- 停止服务。
```sh
./zkServer.sh stop
```

- 参考：
  - [linux安装zk](https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/index.html)。
  - [zk命令行客户端使用](https://blog.csdn.net/ganglia/article/details/11606807)。

### kafka ###
- 下载解压kafka:[https://kafka.apache.org/downloads.html](https://kafka.apache.org/downloads.html)
- 创建文件夹、配置server。
```sh
cd /usr/local/kafka/kafka_2.12-2.5.0/config
vim server.properties
```
```txt
broker.id=1
port=9092
host.name=172.16.81.77
advertised.host.name=47.114.62.101
advertised.listeners=PLAINTEXT://47.114.62.101:9092

num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/usr/local/kafka/kafka_2.12-2.5.0/kafka-logs
num.partitions=2
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

zookeeper.connect=localhost:2181/kafka1
zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=0
```

- 启动kafka。
```sh
cd /usr/local/kafka/kafka_2.12-2.5.0/bin
./kafka-server-start.sh -daemon ../config/server.properties
```

- 创建topic。
```sh
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic dblab01
./kafka-topics.sh --describe --zookeeper localhost:2181 --topic dblab01
./kafka-topics.sh --list --zookeeper localhost:2181
```

- 生产和消费消息。[详细测试](https://www.cnblogs.com/zyt-bg/p/10368786.html)
```sh
./kafka-console-producer.sh --broker-list localhost:9092 --topic dblab01
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic dblab01 --from-beginning
```

### kafka在zk中的节点 ###
- [参考](https://juejin.im/post/5eef7ab6f265da02cf498fb9)：<br>![](https://user-gold-cdn.xitu.io/2020/6/21/172d77752dad210d)

## 使用 ##
- 引入依赖。
```java
compile 'org.apache.kafka:kafka_2.12:2.5.0'
```

### 生产者使用 ###
- 生产者配置。
```java
    @Bean
    public KafkaProducer<String, String> kafkaProducer() {
        Properties pros = new Properties();
        pros.put("bootstrap.servers", "xxxxxx:9092");
        pros.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        pros.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        pros.put("acks", "-1");
        return new KafkaProducer<>(pros);
    }

```

- 同步发送和异步发送。
```java
    private void produceSync() {
        for (int i = 0; i < 3; i++) {
            try {
                ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC, "key_" + i, "val_" + i);
                kafkaProducer.send(record).get();
                System.out.println("send successful: " + record.key());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private void produceAsync() {
        for (int i = 10; i < 13; i++) {
            try {
                ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC, "key_" + i, "async_val_" + i);
                kafkaProducer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        exception.printStackTrace();
                    } else {
                        System.out.println("send successful: " + record.key());
                    }
                });
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```

### 消费者使用 ###
- 消费者配置。
```java
    @Bean
    public KafkaConsumer<String, String> kafkaConsumer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "xxxxxx:9092");
        props.put("group.id", "consumerGroup2");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("auto.offset.reset", "earliest");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        return new KafkaConsumer<>(props);
    }
```

- 三种提交方式。
```java
    public void consume() {
        //订阅主题
        kafkaConsumer.subscribe(Lists.newArrayList(TOPIC));
        new Thread(() -> {
            //轮询消费
            Duration duration = Duration.ofSeconds(1);
            while (true) {
                ConsumerRecords<String, String> records = kafkaConsumer.poll(duration);
                for (ConsumerRecord<String, String> record : records) {
                    System.out
                            .printf("offset = %d ,key =%s, value= %s%n", record.offset(), record.key(), record.value());
                    System.out.println();
                }
                //自动提交、同步提交、异步提交三选一
                kafkaConsumer.commitSync();
                kafkaConsumer.commitAsync();
            }
        }).start();
    }
```