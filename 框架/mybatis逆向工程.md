[toc]
## 简介 ##
- 通过配置文件生成数据库对应的 **实体类、dao接口、mapper的xml文件**。
- 避免大同小异dao层的编写。
- 无法进行联结查询等较高级的操作。
- ![](http://img-blog.csdn.net/2018031921161990)

## 生成需要的文件 ##
### 环境搭建 ###
- 创建简单java工程。
- 导入lib中的5个jar包。
- 配置log4j.properties。例如：
```xml
log4j.rootLogger=DEBUG, Console
#Console
log4j.appender.Console=org.apache.log4j.ConsoleAppender
log4j.appender.Console.layout=org.apache.log4j.PatternLayout
log4j.appender.Console.layout.ConversionPattern=%d [%t] %-5p [%c] - %m%n
log4j.logger.java.sql.ResultSet=INFO
log4j.logger.org.apache=INFO
log4j.logger.java.sql.Connection=DEBUG
log4j.logger.java.sql.Statement=DEBUG
log4j.logger.java.sql.PreparedStatement=DEBUG
```

### 逆向工程配置文件 ###
- 配置generatorConfig.xml。注意：
    - jdbcConnection中的**数据库连接**信息。
    - javaModelGenerator中配置**实体类**包名。
    - sqlMapGenerator中配置**mapper的xml文件** 的包名。
    - javaClientGenerator中配置 **dao接口** 的包名。
    - table中指定需要使用逆向工程的**数据库表**。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
  <context id="testTables" targetRuntime="MyBatis3">
    <commentGenerator>
      <!-- 是否去除自动生成的注释 -->
      <property name="suppressAllComments" value="true"/>
    </commentGenerator>
    <!--数据库连接的信息：驱动类、连接地址、用户名、密码 -->
    <jdbcConnection driverClass="com.mysql.jdbc.Driver"
      connectionURL="jdbc:mysql://localhost:3306/taotao" userId="root"
      password="root">
    </jdbcConnection>
    <!-- 默认false，把JDBC DECIMAL 和 NUMERIC 类型解析为 Integer，为 true时把JDBC DECIMAL 和
      NUMERIC 类型解析为java.math.BigDecimal -->
    <javaTypeResolver>
      <property name="forceBigDecimals" value="false"/>
    </javaTypeResolver>

    <!-- targetProject:生成PO类的位置 -->
    <javaModelGenerator targetPackage="com.taotao.pojo"
      targetProject=".\src">
      <!-- enableSubPackages:是否让schema作为包的后缀 -->
      <property name="enableSubPackages" value="false"/>
      <!-- 从数据库返回的值被清理前后的空格 -->
      <property name="trimStrings" value="true"/>
    </javaModelGenerator>
    <!-- targetProject:mapper映射文件生成的位置 -->
    <sqlMapGenerator targetPackage="com.taotao.mapper"
      targetProject=".\src">
      <!-- enableSubPackages:是否让schema作为包的后缀 -->
      <property name="enableSubPackages" value="false"/>
    </sqlMapGenerator>
    <!-- targetPackage：mapper接口生成的位置 -->
    <javaClientGenerator type="XMLMAPPER"
      targetPackage="com.taotao.mapper"
      targetProject=".\src">
      <!-- enableSubPackages:是否让schema作为包的后缀 -->
      <property name="enableSubPackages" value="false"/>
    </javaClientGenerator>
    <!-- 指定数据库表 -->
    <table schema="" tableName="tb_content"></table>
    <table schema="" tableName="tb_content_category"></table>
    <table schema="" tableName="tb_item"></table>
    <table schema="" tableName="tb_item_cat"></table>
    <table schema="" tableName="tb_item_desc"></table>
    <table schema="" tableName="tb_item_param"></table>
    <table schema="" tableName="tb_item_param_item"></table>
    <table schema="" tableName="tb_order"></table>
    <table schema="" tableName="tb_order_item"></table>
    <table schema="" tableName="tb_order_shipping"></table>
    <table schema="" tableName="tb_user"></table>

  </context>
</generatorConfiguration>
```

### 运行逆向工程 ###
- 编写启动类Generator，运行工程。
```java
public class Generator {

    public void generator() throws Exception {

        List<String> warnings = new ArrayList<String>();
        //允许覆盖
        boolean overwrite = true;
        //配置文件
        File configFile = new File("generatorConfig.xml");
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(configFile);
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);

        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config,
                callback, warnings);
        myBatisGenerator.generate(null);

    }

    public static void main(String[] args) throws Exception {
        try {
            Generator generator = new Generator();
            generator.generator();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

### 生成文件位置 ###
- ![mybatisGenerator2.png](http://img-blog.csdn.net/2018031921344268)

## 使用生成的类 ##
- 使用dao接口-mapper访问数据库。
    - 通过主键操作。
    - 通过example,添加查询条件操作。
```java
    public TbItem getItemById(long itemId) {
		TbItem item = itemMapper.selectByPrimaryKey(itemId);
		return item;
    }
```
```java
    public TbItem getItemById(long itemId) {
        //添加查询条件
        TbItemExample example = new TbItemExample();
        Criteria criteria = example.createCriteria();
        criteria.andIdEqualTo(itemId);
        List<TbItem> list = itemMapper.selectByExample(example);
        if (list != null && list.size() > 0) {
            TbItem item = list.get(0);
            return item;
        }
        return null;
    }
```

## 相关链接 ##
- 文档：[http://www.mybatis.org/generator/index.html](http://www.mybatis.org/generator/index.html)
- 最新项目地址：[https://github.com/mybatis/generator](https://github.com/mybatis/generator)
- 一个可以直接使用的逆向工程：[https://github.com/hanjg/mybatisGenerator](https://github.com/hanjg/mybatisGenerator)