## Nacos简介 ##
- [nacos简介](https://blog.csdn.net/qq_40369829/article/details/108945326)。

## Server ##
- 下载最新版本、启动server。[参考文档](https://nacos.io/zh-cn/docs/quick-start.html)。
- 默认的本地admin地址：http://127.0.0.1:8848/nacos

## Client ##
### 引入依赖 ###
- house-common中引入nacos依赖。
```xml
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
```


### 应用上下文配置外部化 ###
- house-api为例。
- ```bootstrap.properties```配置如下。
- 如HOUSE_ENV=dev，则加载 house-api-application.yaml， house-base-mysql.yaml 。[约定参考](https://nacos.io/zh-cn/docs/quick-start-spring-cloud.html)。
```properties
spring.application.name=house-api
spring.profiles.active=api-application,base-mysql
spring.cloud.nacos.config.namespace=${HOUSE_ENV}
spring.cloud.nacos.config.group=house
spring.cloud.nacos.config.server-addr=${NACOS_ADDR}
spring.cloud.nacos.config.prefix=house
spring.cloud.nacos.config.file-extension=yaml
```

- nacos admin页面创建namespace。<br>![200915.naocs.namespace.png](https://img-blog.csdnimg.cn/20201020130109516.png)
- 创建配置。<br>![200915.naocs.dataid.png](https://img-blog.csdnimg.cn/20201020130109676.png)
- house-api-application.yaml:
```yaml
server:
  port: 8081
  servlet:
    context-path: /
```

- house-base-mysql.yaml。
```yaml
spring:
  datasource:
    name: development
    url: jdbc:mysql://localhost:3306/house?characterEncoding=utf8
    username: root
    password: root
    # 使用druid数据源
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    filters: stat
    maxActive: 20
    initialSize: 1
    maxWait: 60000
    minIdle: 1
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: select 'x'
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true
    maxOpenPreparedStatements: 20

mybatis:
  mapper-locations: classpath:mapper/*.xml  #对应mapper映射xml文件的所在路径
#  type-aliases-package: com.babyjuan.house.repository.mysql.entity # 对应实体类的路径非必须

#pagehelper分页插件
pagehelper:
  helperDialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```

> 注意：**Nacos配置带中文**，[需要启动参数加上utf-8编码](https://www.jianshu.com/p/81e0f5b78adf)。
```bat
java  -Dfile.encoding=utf-8 -jar .\house-api-0.0.1-SNAPSHOT-exec.jar 
``` 

### 业务配置外部化 ###
- house-common中实现初始化和监听更新的Nacos配置父类。
```java
public abstract class BaseNacoConfig {

    protected Logger logger = LoggerFactory.getLogger(getClass());

    @Value("${spring.cloud.nacos.config.server-addr}")
    private String serverAddr;

    @Value("${spring.cloud.nacos.config.namespace}")
    private String namespace;

    @Value("${spring.cloud.nacos.config.group}")
    private String group;

    private ConfigService configService;

    @PostConstruct
    public void init() throws Exception {
        Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
        properties.put(PropertyKeyConst.NAMESPACE, namespace);
        configService = NacosFactory.createConfigService(properties);

        initContent(fetchContent());

        configService.addListener(dataId(), group, new Listener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                try {
                    changeContent(fetchContent());
                } catch (NacosException e) {
                    logger.error("{} change error.", dataId());
                }
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        });
    }

    private String fetchContent() throws NacosException {
        String content = configService.getConfig(dataId(), group, 5000);
        logger.info("fetch data, dataId: {}, content: {}", dataId(), content);
        return content;
    }

    public abstract String dataId();

    public abstract void initContent(String content);

    public abstract void changeContent(String content);

}
```

- house-task中监听开关的变更，控制运行。
```java
@Configuration
public class CrawlerSwitchConfig extends BaseNacoConfig {

    private volatile boolean on;

    @Resource(name = "secondHandCrawlerCommandImpl")
    private CrawlerCommand crawlerCommand;

    @Override
    public String dataId() {
        return "house-task-spider-switch";
    }

    @Override
    public void initContent(String content) {
        on = BooleanUtils.toBoolean(content);
    }

    @Override
    public void changeContent(String content) {
        boolean newStatus = BooleanUtils.toBoolean(content);
        if (newStatus && !on) {
            crawlerCommand.start(1);
        } else if (!newStatus && on) {
            crawlerCommand.stop();
        } else {
            logger.info("crawler status unchange, {} -> {}", on, newStatus);
        }
        on = newStatus;
    }

    public boolean on() {
        return on;
    }
}
```

```java
@Component
public class SpiderTask {

    protected Logger logger = LoggerFactory.getLogger(getClass());

    @Resource(name = "secondHandCrawlerCommandImpl")
    private CrawlerCommand secondHandCrawlerCommand;

    @Resource(name = "shHouseDealCrawlerCommandImpl")
    private CrawlerCommand shHouseDealCrawlerCommand;

    @Autowired
    private CrawlerSwitchConfig crawlerSwitchConfig;

    @Scheduled(fixedDelay = 3 * 1000)
    public void shHouse() {
        if (!crawlerSwitchConfig.on()) {
            return;
        }
        secondHandCrawlerCommand.start(1);
    }
}
```

### Dubbo配置外部化 ###
- [参考](https://dubbo.apache.org/zh-cn/docs/user/configuration/annotation.html)。
- house-api,house-rest为例。

#### provider ####
- house-api-provider中的dubbo配置移到house-api-application.yaml。
```xml
server:
  port: 8081
  servlet:
    context-path: /

dubbo:
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    port: 28081
  registry:
    address: zookeeper://127.0.0.1:2181
    timeout: 60000
    protocol: zookeeper
```

- 接口实现改用dubbo注解暴露。
```java
import com.alibaba.dubbo.config.annotation.Service;
@Service
public class SecondHandHouseServiceImpl implements SecondHandHouseService {
}
```

- 指定扫描路径。
```java
@SpringBootApplication
@EnableDubbo(scanBasePackages = "com.babyjuan.house.api.service")
public class HouseApiApplication implements WebMvcConfigurer {
}
```

#### consumer ####
- house-rest-consumer中的dubbo配置移到house-rest-application.yaml。
```yaml
server:
  port: 8080
  servlet:
    context-path: /house
    
dubbo:
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    port: 28080
  registry:
    protocol: "zookeeper"
    address: zookeeper://127.0.0.1:2181
    timeout: "60000"
  consumer:
    check: false
```

- 使用注解引用dubbo接口。
```java
import com.alibaba.dubbo.config.annotation.Reference;
public class SecondHandHouseController {
    @Reference
    private SecondHandHouseService secondHandHouseService;
}
```

- 开启dubbo注解。
```java

@EnableDubbo
public class HouseGatewayApplication implements WebMvcConfigurer {
}
```