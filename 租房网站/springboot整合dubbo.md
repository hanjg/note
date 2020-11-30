[toc]
## dubbo原理 ##
- [dubbo原理和特点](https://blog.csdn.net/qq_40369829/article/details/107774420)

## 多模块项目创建 ##
- [多模块创建参考](https://blog.csdn.net/liyanlei5858/article/details/79047884)。
- 各个模块依赖关系：<br>![201021.house.module.png](https://img-blog.csdnimg.cn/20201021010729529.png)
- 父模块管理依赖版本，不直接依赖。
- [打包注意](https://blog.csdn.net/u012702547/article/details/95180256)：
  - springboot默认打包成可执行jar，无法被依赖。
  - 打包插件里需要重命名jar包，原文件才不会被重命名，可依赖。
```xml
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
          <!--重命名可执行jar-->
          <classifier>exec</classifier>
        </configuration>
      </plugin>
    </plugins>
```

## provider ##
- house-api服务为例。

### 接口定义模块 ###
- house-contract：定义dubbo接口，无额外依赖。
```java
package com.babyjuan.house.contract.service;

public interface SecondHandHouseService {
    BaseResponse<List<String>> getAllDistricts();
}
```

### 公共模块 ###
- house-common：引入接口定义、web应用、dubbo需要的依赖+通用工具类。
```java
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-contract</artifactId>
      <version>${house.version}</version>
    </dependency>

    <!--web-->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <version>${spring.boot.version}</version>
      <exclusions>
        <!--log4j和logback冲突，干掉logback-->
        <exclusion>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>

    <!--rpc-->
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>dubbo</artifactId>
    </dependency>
    <dependency>
      <groupId>io.netty</groupId>
      <artifactId>netty-all</artifactId>
    </dependency>
    <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-framework</artifactId>
    </dependency>

    <!--utils-->
    <dependency>
      <groupId>commons-beanutils</groupId>
      <artifactId>commons-beanutils</artifactId>
    </dependency>
    <dependency>
      <groupId>joda-time</groupId>
      <artifactId>joda-time</artifactId>
    </dependency>
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-lang3</artifactId>
    </dependency>
```

### 存储模块 ###
- house-repository：引入公共模块和存储依赖，访问db等存储。
```java
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-common</artifactId>
      <version>${house.version}</version>
    </dependency>

    <!--dao-->
    <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <scope>runtime</scope>
    </dependency>
    <!-- 分页插件 -->
    <dependency>
      <groupId>com.github.pagehelper</groupId>
      <artifactId>pagehelper-spring-boot-starter</artifactId>
    </dependency>
    <!-- alibaba的druid数据库连接池 -->
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid-spring-boot-starter</artifactId>
    </dependency>
```

- 实现db访问。
```java
package com.babyjuan.house.repository;

public interface MysqlRepository {
    List<String> queryAllDistricts();
}
```

### 查询模块 ###
- house-api：依赖存储模块，提供查询接口。
```java
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-repository</artifactId>
      <version>${house.version}</version>
    </dependency>
```

- 实现dubbo接口定义。
```java
package com.babyjuan.house.api.service;

@Service
public class SecondHandHouseServiceImpl implements SecondHandHouseService {

    @Autowired
    private MysqlRepository mysqlRepository;

    @Override
    public BaseResponse<List<String>> getAllDistricts() {
        List<String> districts = mysqlRepository.queryAllDistricts();
        return BaseResponse.newSuccessResponse(districts);
    }
}
```

- 在```house-api-provider.xml``` 暴露接口实现。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
   http://dubbo.apache.org/schema/dubbo
   http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

  <!-- 提供方应用信息，用于计算依赖关系 -->
  <dubbo:application name="house-api"/>
  <!-- 使用zookeeper注册中心暴露服务地址 -->
  <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181" timeout="60000"/>
  <!-- 用dubbo协议在28081端口暴露服务 -->
  <dubbo:protocol name="dubbo" port="28081"/>
  <!-- 声明需要暴露的服务接口 -->
  <dubbo:service interface="com.babyjuan.house.contract.service.SecondHandHouseService" ref="secondHandHouseServiceImpl"/>

</beans>
```

## consumer ##
- house-rest服务。

### rpc适配模块 ###
- house-integration：依赖公共模块，调用其他dubbo服务。
```xml
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-common</artifactId>
      <version>${house.version}</version>
    </dependency>
```

- 在``` house-consumer.xml``` 申明需要调用的服务。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
   http://dubbo.apache.org/schema/dubbo
   http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

  <!-- 提供方应用信息，用于计算依赖关系 -->
  <dubbo:application name="house-rest"/>
  <!-- 使用zookeeper注册中心暴露服务地址 -->
  <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181" timeout="60000"/>
  <!--关闭服务消费方所有服务的启动检查。dubbo缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止Spring初始化完成。-->
  <dubbo:consumer check="false"/>
  <!-- 用dubbo协议在28080端口暴露服务 -->
  <dubbo:protocol name="dubbo" port="28080"/>
  <!-- 使用xml配置方式创建远程服务代理，id即为provider.xml中暴露的服务的id-->
  <dubbo:reference id="secondHandHouseService" interface="com.babyjuan.house.contract.service.SecondHandHouseService"/>
</beans>
```

### web模块 ###
- house-rest：引入适配器模块，提供http接口。
```xml
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-integration</artifactId>
      <version>${house.version}</version>
    </dependency>
```

- 将rpc接口转为http接口暴露。
```java
package com.babyjuan.house.gateway.controller;

@Controller
@RequestMapping("/secondHandHouse")
public class SecondHandHouseController {

    @Autowired
    private SecondHandHouseService secondHandHouseService;

    @RequestMapping("/districts")
    @ResponseBody
    public BaseResponse<List<String>> getAllDistricts() {
        return secondHandHouseService.getAllDistricts();
    }
}
```