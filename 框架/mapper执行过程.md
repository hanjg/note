[toc]
## mapper的初始化流程 ##
### spring容器的启动 ###
- SpringApplication.run中的refreshContext方法，实现bean的实例化和装配，包括spring.factories的加载。
- [springboot自动化配置](https://www.cnblogs.com/trgl/p/7353782.html)。

### 代理类工厂的初始化 ###
- spring容器启动时，由MybatisAutoConfiguration创建sqlSessionFactory，解析mapper，并在MapperRegistry中持有mapper和代理类工厂的映射。<br>![190702.init.png](https://img-blog.csdnimg.cn/20190709102241680.png)

### 代理实例的初始化 ###
- 自动装配mapper对象时，mapper的代理类由mapperProxyFactory创建(jdk动态代理）。<br>![190702.mapper.png](https://img-blog.csdnimg.cn/20190702224905716.png)

## mapper的执行流程 ##
- 一次select流程。<br>![190702.exe.png](https://img-blog.csdnimg.cn/20190702224303336.png)

## 源码分析文件 ##
- [mybatis架构](https://blog.csdn.net/luanlouis/article/details/40422941)。
- [mybatis深入浅出](https://my.oschina.net/xianggao/blog/548873)。
- [springboot集成mybatis](https://www.cnblogs.com/nxzblogs/tag/mybatis/)。
- [datasource的加载过程](https://www.cnblogs.com/storml/p/8611388.html)。