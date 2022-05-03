
@[toc]
## 特点 ##
1. sql与代码的分离。
    - 优点：便与管理和维护。
    - 缺点：不便于调试，需要借助日志，并设置等级为debug>
2. 标签控制动态sql拼接。
    - 优点：标签替代逻辑代码，清晰。
    - 缺点：拼接复杂sql时，没有代码灵活。
3. 结果集和java对象自动映射。
    - 优点：保证名称相同即可映射。
    - 缺点：对开发的sql依赖性强。
4. 编写原生sql。
    - 优点：接近jdbc，灵活。
    - 确定：对sql依赖度高，半自动，数据库移植不便。 

## 数据库会话 ##
- sqlSession的作用。
    - 向sql语句传入参数。
    - 执行sql语句。
    - 获取执行sql语句的结果。
    - 事务控制。
- 打开方式：
```java
    public SqlSession getSqlSession() throws IOException {
        //读取数据库连接信息
        Reader reader = Resources.getResourceAsReader("mybatis/Configuration.xml");
        //通过连接信息获得SqlSessionFactory
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        //打开数据库会话
        return sqlSessionFactory.openSession();
    }
```
- sqlSession只能传递**一个对象**，可以通过封装类和MAP传递多个参数。
- mybatis通过sqlSession的**增删改**默认**手动提交**。
```java
sqlSession.commit();
```

## 基本配置 ##
- 基本配置可以参考[http://blog.csdn.net/qq_40369829/article/details/79323270#dao层](http://blog.csdn.net/qq_40369829/article/details/79323270#dao层)，和如下。
- dao层方法和dao接口：
```java
    public List<Message> queryMessageList(Map<String, Object> parameters) {
        List<Message> messageList = new ArrayList<Message>();
        SqlSession sqlSession = null;
        try {
            sqlSession = dbAccess.getSqlSession();
            IMessage iMessage = sqlSession.getMapper(IMessage.class);
            messageList = iMessage.queryMessageList(parameters);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (sqlSession != null) {
                sqlSession.close();
            }
        }
        return messageList;
    }
```
```java
public interface IMessage {
    List<Message> queryMessageList(Map<String, Object> parameters);
}
```
- dao接口对应的sql语句配置。
    - namespace为包名.接口类名。
    - id为接口方法。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.hjg.mybatis.mircoMessage.dao.IMessage">
  <resultMap id="messageResult" type="com.hjg.mybatis.mircoMessage.bean.Message">
    <!--id标签为主键列，result标签为一般列-->
    <id column="id" jdbcType="INTEGER" property="id"/>
    <result column="command" jdbcType="VARCHAR" property="command"/>
    <result column="description" jdbcType="VARCHAR" property="description"/>
    <result column="content" jdbcType="VARCHAR" property="content"/>
  </resultMap>

  <sql id="columns">id, command, description, content</sql>

  <select id="queryMessageList" parameterType="java.util.Map" resultMap="messageResult">
    SELECT
    <include refid="columns"/>
    FROM message
    <where>
      <if test="message.command != null and !&quot;&quot;.equals(message.command.trim())">AND command =
        #{message.command}
      </if>
      <if test="message.description != null and !&quot;&quot;.equals(message.description.trim())">AND description LIKE
        '%'
        #{message.description}
        '%'
      </if>
    </where>
    ORDER BY id LIMIT #{page.dbLimit} OFFSET #{page.dbOffset}
  </select>
</mapper>
```
- mybatis核心xml：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <!-- 对事务的管理和连接池的配置 -->
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="UNPOOLED">
        <property name="driver" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/micromessage"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="mybatis/sql/Message.xml"/>
  </mappers>
</configuration>
```

## OGNL表达式 ##
- OGNL表达式，用于向sql配置文件传递参数,使用#{}取值填充参数，也可以使用java语法（如equals等）。语法如下：<br>![ognl.png](https://img-blog.csdnimg.cn/img_convert/648a05a480cf7a8c3db9825a8b524f62.png)<br>![ognl2.png](https://img-blog.csdnimg.cn/img_convert/d14d20e846c1f6f385ef733aa3724094.png)
- 可以使用separator属性，分隔集合中的元素。
```xml
  <delete id="deleteBatch" parameterType="java.util.List">
    DELETE FROM message WHERE id IN (
    <foreach collection="list" item="item" separator=",">
      #{item}
    </foreach>
    )
  </delete>
```
- 常用标签。<br>![mark.png](https://img-blog.csdnimg.cn/img_convert/bcf0d58721e65bdfcb231d32bd8c5ab0.png)
    - where标签：标签中的都不满足，不添加where关键字。满足则进行格式处理，如删除第一个and。
    - set标签类似where标签，用于update。
    - trim标签,格式字符。
    - sql标签定义常量。
    - choose标签可以进行ifelse逻辑。
    - association标签提供关联查询。
```xml
  <sql id="columns">id,command,description,content</sql>
  <select id="queryMessageList" parameterType="com.hjg.mybatis.mircoMessage.bean.Message" resultMap="messageResult">
    SELECT
    <include refid="columns"/>
    FROM message
    <where>
      <if test="command != null and !&quot;&quot;.equals(command.trim())">AND command = #{command}</if>
      <if test="description != null and !&quot;&quot;.equals(description.trim())">AND description LIKE '%'
        #{description}
        '%'
      </if>
      <choose>
        <when test=""></when>
        <when test=""></when>
        <otherwise></otherwise>
      </choose>
    </where>

    #代替where
    <trim prefix="where" prefixOverrides="and/or"></trim>
    #代替set
    <trim prefix="set" suffixOverrides=","></trim>
  </select>
```

## 原理 ##
### 易混淆的概念 ###
- resultMap,resultType。后者是前者的阉割版，根据类的属性名和查询的列对应，大小写不敏感。前者还可以用typeHandler属性进行类型转换。	
- parameterType可以根据OGNI表达式取值。parameterMap**不推荐**使用。
- #{}有预编译效果，相当于替换为？再赋值。${}直接进行替换，例如用在order by ${}。${}不宜过度使用。

### 乱码问题 ###
- 可能乱码的位置：
    - 文件本身
    - jsp设置编码
    - servlet串值时编码
    - get方式提交中文时Tomcat编码
    - 数据库编码（可以在数据库Url传参数，设定编码）
    - 数据库表编码。

### 接口式编程 ###
- 定义dao接口，通过xml配置sql。
- 通过**动态代理**实现。
- [原理](https://www.imooc.com/video/5896)

### mapper的初始化流程 ##
#### 代理类工厂的初始化 ####
> [springboot基于spring.factories的自动装配](https://blog.csdn.net/qq_40369829/article/details/84753616)。
1. MybatisAutoConfiguration配置在spring.factories文件中，并在容器启动时加载bean的定义。<br>![在这里插入图片描述](https://img-blog.csdnimg.cn/61d5fc8549c24cc185ffc9c222880d59.png)
2. 容器refresh时在finishBeanFactoryInitialization中创建sqlSessionFactory这个bean
	1. 创建bean的时候读取mapper的xml配置，位置：org.mybatis.spring.SqlSessionFactoryBean#buildSqlSessionFactory
	2. 在MapperRegistry中解析mapper，创建代理类工厂MapperProxyFactory，位置：org.apache.ibatis.binding.MapperRegistry#addMapper

#### 代理实例的初始化 ####
- 依赖mapper的bean初始化时，使用jdk动态代理创建mapper代理类，位置：org.apache.ibatis.binding.MapperProxyFactory#newInstance(org.apache.ibatis.session.SqlSession)

### mapper的执行流程 ###
- 一次select流程。<br>![190702.exe.png](https://img-blog.csdnimg.cn/20190702224303336.png)
- 参考。
	- [mybatis架构](https://blog.csdn.net/luanlouis/article/details/40422941)。
	- [mybatis深入浅出](https://my.oschina.net/xianggao/blog/548873)。
	- [springboot集成mybatis](https://www.cnblogs.com/nxzblogs/tag/mybatis/)。
	- [datasource的加载过程](https://www.cnblogs.com/storml/p/8611388.html)。

## 实例 ##
### 获取自增主键值 ###
- [参考](https://blog.csdn.net/zhenwodefengcaii/article/details/73195902)。

#### 不返回值 ####
- 使用useGeneratedKeys，keyProperty两个属性获得自增的主键值。在完成插入之后赋给对象对应的属性（这个id插入之前不存在）。
```xml
  <insert id="insertCommand" useGeneratedKeys="true" keyProperty="id"
    parameterType="com.hjg.mybatis.mircoMessage.bean.Command">
    INSERT INTO command (command, description) VALUES (#{command}, #{description})
  </insert>
```

#### 返回值 ####
- 使用selectKey标签和SELECT LAST_INSERT_ID()语句。
```xml
  <insert id="insertSelective" parameterType="com.babyjuan.house.dao.entity.Community" >
    <selectKey keyProperty="infoId" resultType="long" order="AFTER">
      SELECT LAST_INSERT_ID()
    </selectKey>
    insert into community
  ...
```

### 分页拦截器 ###
- 需要实现 org.apache.ibatis.plugin.Interceptor 接口。
- 需要熟悉mybatis源码。
```java
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class})})
public class PageInterceptor implements Interceptor {
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = MetaObject.forObject(statementHandler, SystemMetaObject.DEFAULT_OBJECT_FACTORY,
                DEFAULT_OBJECT_WRAPPER_FACTORY, new DefaultReflectorFactory());
        MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        // 配置文件中SQL语句的ID
        String id = mappedStatement.getId();
        if (id.matches(".+ByInterceptor$")) {
            BoundSql boundSql = statementHandler.getBoundSql();
            // 原始的SQL语句
            String sql = boundSql.getSql();
            // 查询总条数的SQL语句
            String countSql = "select count(*) from (" + sql + ")a";
            Connection connection = (Connection) invocation.getArgs()[0];
            PreparedStatement countStatement = connection.prepareStatement(countSql);
            ParameterHandler parameterHandler = (ParameterHandler) metaObject.getValue("delegate.parameterHandler");
            parameterHandler.setParameters(countStatement);
            ResultSet rs = countStatement.executeQuery();

            Map<?, ?> parameter = (Map<?, ?>) boundSql.getParameterObject();
            Page page = (Page) parameter.get("page");
            if (rs.next()) {
                page.setTotalNumber(rs.getInt(1));
            }
            // 改造后带分页查询的SQL语句
            String pageSql = sql + " limit " + page.getDbLimit() + " offset " + page.getDbOffset();
            metaObject.setValue("delegate.boundSql.sql", pageSql);
        }
        return invocation.proceed();
    }

    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    /**
     * 获取配置文件中的属性值
     */
    public void setProperties(Properties properties) {
    }

}
```

### 批量新增 ###
- 在sql中传入list参数，使用ognl表达式foreach拼接sql。
```xml
  <insert id="insertBatch" parameterType="java.util.List">
    INSERT INTO command_content(content, commandId) VALUES
    <foreach collection="list" item="item" separator=",">
      (#{item.content},#{item.commandId})
    </foreach>
  </insert>
```

## 坑 ##
### 泛型擦除 ###
- 调用mapper返回 GoodsExtendedAttribute 对象列表。编译不报错，运行报错。
```xml
<select id="selectGoodsIdByExample" resultMap="BaseResultMap" parameterType="xxxx.GoodsExtendedAttributeExample">
    select goods_id from goods_extended_attribute
    <if test="_parameter != null">
      <include refid="Example_Where_Clause"/>
    </if>
    <if test="orderByClause != null">
      order by ${orderByClause}
    </if>
  </select>
```
```java
List<Long> list =  goodsExtendedAttributeMapper.selectGoodsIdByExample(goodsExample);
```

### 资源路径 ###
- IDEA resource中显示的包路径对应实际目录可能有两种：
  - D:\project\gin\gin\gin-repository\src\main\resources\base\com\yiran\service\gin\repository
  - D:\project\gin\gin\gin-repository\src\main\resources\base\com.yiran.service.gin.repository
- ![210805.mybatis.png](https://img-blog.csdnimg.cn/1077adef62c8439bbb4a30b8b4c3963b.png)
## 参考 ##
- [https://mp.weixin.qq.com/s/_W2K9bv6EQqBg6qiwUE0jQ](https://mp.weixin.qq.com/s/_W2K9bv6EQqBg6qiwUE0jQ)
