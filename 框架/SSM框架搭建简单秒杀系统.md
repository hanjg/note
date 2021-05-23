[toc]
## 简介 ##
- 根据慕课网教程整合SSM框架编写秒杀系统，添加增删改和分页查询功能，并使用cdn、服务器缓存、存储过程进行高并发优化。
- 视频地址：[http://www.imooc.com/course/programdetail/pid/59](http://www.imooc.com/course/programdetail/pid/59)
- 源码地址：[https://github.com/hanjg/seckill_ssm](https://github.com/hanjg/seckill_ssm)

### 高并发优化 ###
- 前端控制。如暴露秒杀接口时防止重复按键。
- 数据缓存。
    - 使用cdn缓存分发静态资源，如js,css等。
    - 使用redis缓存商品信息，暴露秒杀接口,因其一致性无法使用cdn并且一致性维护成本低。
- 减少行级锁的持有时间。
    - 优先插入购买明细，再更新商品信息，从逻辑上减小商品行级锁的持有时间。
    - 使用**存储过程**，将事务操作移到数据库，可以减少网络延迟和GC，减小行级锁的持有时间。简单的逻辑可以应用存储过程，但**不能过度依赖**。

### 分页 ###
- 使用后端分页，根据查询参数动态的从数据库获得信息，减小前后端通信的数据量。

## 搭建 ##
### 环境 ###
#### maven安装 ####
- maven下载地址：[http://maven.apache.org/download.cgi](http://maven.apache.org/download.cgi "http://maven.apache.org/download.cgi")
- 将bin文件夹配置到环境变量。
- idea中配置maven。配置地点和maven的setting文件主要配置参考如下:<br>![mavensetting.png](http://img.blog.csdn.net/20180212215908150)<br>
```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>A:/java/repository</localRepository>
  <profiles>
    <profile>
      <id>jdk-1.8</id>
      <activation>
        <jdk>1.8</jdk>
      </activation>
      <repositories>
        <repository>
          <id>jdk18</id>
          <name>Repository for JDK 1.8 builds</name>
          <!--<url>http://www.myhost.com/maven/jdk18</url> -->
          <url>http://central.maven.org/maven2/</url>
          <layout>default</layout>
        </repository>
      </repositories>
    </profile>
  </profiles>
</settings>
```

#### Redis安装 ####
- 下载redis：[https://github.com/MicrosoftArchive/redis/releases](https://github.com/MicrosoftArchive/redis/releases)
- redis-server.exe启动redis。
- 程序中通过jedis和protostuff使用redis。

### 工程基本配置 ###
- 创建maven项目，选择webapp模板。<br>![maven.png](http://img.blog.csdn.net/20180212213044698)
- 在Project Structure创建目录。<br>![struct.png](http://img.blog.csdn.net/20180212213630220)<br>![struct3.png](http://img.blog.csdn.net/20180212214851160)
- 配置pom.xml，包含需要的jar包。
    - slf4j是接口规范，logback/log4j/common-logging等是日志实现，此处使用log4j的一个改良版本logback。
    - spring并未提供mybatis依赖，mybatis通过mybatis-spring实现依赖。spring的版本号为 4.3.13.**RELEASE**。
    - protostuff是一种序列化工具，性能优于jdk序列化。

## 数据库 ##
- 创建mysql数据库seckill，并且通过schema.sql创建和初始化表。
    - sql的 ```Invalid default value for 'timestamp'``` 错误解决方案：
        -  ```SET GLOBAL sql_mode = ''; ```消除对数据的限制。
        -  在jdbc的url后面拼接参数 ```&zeroDateTimeBehavior=convertToNull``` 。 
    - timestamp类型,**范围小**，效率高，**存储标准时间**，可以跨时区。
        - 创建timestamp数据库字段时最好补全限制，否则有可能被默认添加某些类型，如DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP。这样默认值为当前时间，更新数据的时候修改为更新的时间。
        - 如果第一个timeStamp不指定限制，或者只是指定not null,则数据库会自动添加 ```NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP```。
        - [timestamp的用法详解](https://www.cnblogs.com/csl0910/p/4956815.html)。
    - 另外idea的mysql插件显示ddl不靠谱，还是得用navicat一类的图形工具。。。
    - success_killed表中的联合主键可以防止重复插入，即重复秒杀。
```sql
  start_time  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
  COMMENT '秒杀开始时间',
  end_time    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
  COMMENT '秒杀结束时间',
  create_time TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
  COMMENT '创建时间',
```
- 在seckill.sql中编写存储过程seckill.execute_seckill，在数据库端管理秒杀事务。

## DAO层 ##
### 实体 ###
- entity包下开发两张表对应的实体类，Seckill和SuccessKilled。
    - SuccessKilled中需要包含Seckill这个多对一属性，完成秒杀明细和秒杀商品的关联。
    
### 接口 ###
- dao包下开发两张表对应的dao接口，SeckillDao和SuccessKilledDao。
    - 多个参数需要@Param标识指出参数名，否则mybatis无法识别参数名。
    - SuccessKilledDao在使用存储过程管理秒杀事务的时候无须使用。
- dao.cache下开发RedisDao，用于在redis中存取序列化的对象。

### 配置 ###
- 在mybatis-config.xml中配置mybatis全局属性，如列别名替换列名，可以在获得SuccessKilled结果的同时填充seckill属性。
- 在SeckillDao.xml,SuccessDao.xml中配置Dao接口方法对应的sql语句。
    - ```<![CDATA[<=]]>```代替```<=```，否则报错。
    - ``` INSERT IGNORE INTO ``` 。插入时主键冲突时不报错，而是返回0，这样方便业务处理。
- 配置jdbc.properties,提供数据库基本参数。
- 配置spring-dao.xml，包括数据库参数、c3p0数据库连接池、sqlSessionFactory对象、dao接口路径、RedisDao参数。
    - 符合**约定大于配置**的原则：约定某些文件放在某些包下面，自动扫描，而不是个别配置某些文件的路径。

### 单元测试 ###
- 配置logback.xml，用于日志的输出。
- 生成test，配置@RunWith,@ContextConfiguration的属性，编写测试代码。
    - 用calendar获取日期,注意用完之后清零和一月份用0表示。
```java
    private Calendar calendar = Calendar.getInstance();
     calendar.set(2018, 0, 1, 1, 1, 1);
        Date duringDate = calendar.getTime();
        calendar.clear();
```

## Service层 ##
### 实体 ###
- 在dto包下开发Exposer,SeckillExecution,分别存放秒杀暴露的接口、秒杀执行之后的结果。
- 在dto包下开发SeckillResult，用于Controller，操作正确返回结果，否则返回错误信息。
- 在dto包下开发Page，存放分页相关参数。
- 在exception包下开发RepeatKillException，SeckillCloseException，SeckillException。这些**运行时异常**由spring捕获，从而回滚事务。
    - spring只会回滚运行期异常。 
- 在enums包下开发SeckillStateEnum,存放秒杀操作状态，包括状态码和对应信息。

### 接口和实现 ###
- service包下开发SeckillService接口，并在impl下开发实现类SeckillServiceImpl。包括增删改查、暴露秒杀接口的url、服务器端管理秒杀事务、存储过程管理秒杀事务。
    - SeckillServiceImpl类添加注解,包括ioc注解和executeSeckill方法的Transaction注解。

### 配置 ###
- spring-service.xml配置，包括需要扫描注解的包、事务管理器、开启基于注解的声明式事务。

### 单元测试 ###
- 同dao。

## Web层 ##
### Controller开发 ###
- 在web包下开发SeckillController，根据不同的请求url和方法，组织数据，调用service接口。
    - seckill/{seckillId}/{md5}/execution接口可以调用executeSeckillByProcedure通过数据库存储过程执行秒杀，也可以调用executeSeckill通过服务器执行秒杀。前者有利于高并发。
    - @CookieValue required = false，没有对应cookie的时候不会报错。

### 配置 ###
- 配置spring-web.xml,开启注解模式、配置默认servlet-handler、配置默认的ViewResolver和对应的jsp路径、自动扫描的包。
- 配置web.xml，核心的DisptcherServlet。

### 前端 ###
- 前端使用boostrap,页面设计。<br>![front.png](http://img.blog.csdn.net/20180213104931733)
- 详情页的逻辑。<br>![detail.png](http://img.blog.csdn.net/20180213105155860)
- 开发WEB-INF/jsp开发前端页面。
    - list.jsp显示所有商品,detai.jsp显示商品信息和执行秒杀,manager.jsp管理商品，head.jsp和tag.jsp包含通用的引用。
    - seckill.js包括秒杀逻辑，manager.js包括管理商品的逻辑，dateTime.js包括对时间格式化的处理。
    - 注意viewResolver中的路径需要加/，否则调用视图的路径会出错。
	- 静态资源，如boostrap的css和js、某些插件通过bootcdn引入。 
	- 注意引入js时，</script>一定要写，否则之后的内容不加载。
    - js可以使用json封装函数，实现模块化设计。
```jsp
<!-- 新 Bootstrap 核心 CSS 文件 -->
<link href="https://cdn.bootcss.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet">

<!-- jQuery文件。务必在bootstrap.min.js 之前引入 -->
<script src="https://cdn.bootcss.com/jquery/2.1.1/jquery.min.js"></script>
<!-- 最新的 Bootstrap 核心 JavaScript 文件 -->
<script src="https://cdn.bootcss.com/bootstrap/3.3.7/js/bootstrap.min.js"></script>

<%--jquery cookie插件--%>
<script src="https://cdn.bootcss.com/jquery-cookie/1.4.1/jquery.cookie.min.js"></script>
<%--jquery countdown插件--%>
<script src="https://cdn.bootcss.com/jquery.countdown/2.2.0/jquery.countdown.min.js"></script>
```
```jsp
    <script src="<%= basePath %>resources/js/list.js"></script>
    <%--js的</script>一定要写--%>
```

## 部署 ##
- tomcat下载地址：[https://tomcat.apache.org/download-80.cgi](https://tomcat.apache.org/download-80.cgi)
- 在idea中配置tomcat并运行。<br>![tomcat2.png](http://img.blog.csdn.net/2018021312392060)<br>![tomcat.png](http://img.blog.csdn.net/2018021312395330)
- 访问对应url。![listPage.png](http://img.blog.csdn.net/20180213124522383)

