
@[toc]
## 特点 ##
1. 降低组件之间的耦合度，实现软件各层之间的**解耦**。 
2. 可以使用容器提供的众多服务，如：**事务管理服务**、**消息服务**等等。无需手工控制事务，也不需处理复杂的事务传播。
3. 容器提供**单例**模式支持，开发人员不再需要自己编写实现代码。
4. 容器提供了 **AOP** 技术，利用它很容易实现如权限拦截、运行期监控等功能。
5. 容器提供的众多**辅作类**，使用这些类能够加快应用的开发，如： JdbcTemplate、HibernateTemplate。
6. Spring对于主流的应用框架提供了**集成支持**，如：集成Hibernate、JPA、Struts等，这样更便于应用的开发。


## IOC ##
- **控制反转**（Inversion of Control，IOC），应用程序不创建和维护对象，由**外部容器**负责组装。
    - 获得依赖对象的过程被反转了。
- **依赖注入**（Dependency Injection，DI）。将某种依赖关系注入到对象中，IOC的一种实现方式。

### IOC容器 ###
- IOC容器类似**房屋中介**，从容器返回对象（中介介绍房子），程序使用对象（入住）。
- 容器初始化：
    1. 文件： ```new FileSystemXmlApplicationContext（"C:/config.xml"）```。
    2. 类路径： ```new ClassPathXmlApplicationContext（"classpath:spring-config.xml"）```。
    3. Web应用,servlet中配置：
```xml
  <servlet>
    <servlet-name>seckill-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

    <!--配置mvc需要加载的配置文件-->
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring/spring-*.xml</param-value>
    </init-param>
  </servlet>
```

### 依赖注入（DI） ###
- 在启动Spring容器加载bean配置时，完成对变量的赋值。
- 设值注入：
```xml
  <bean id="injectionService" class="com.hjg.spring.ioc.injection.InjectionServiceImpl">
    <property name="injectionDao" ref="injectionDao"></property>
  </bean>
```
- 构造注入：
```xml
  <bean id="injectionService" class="com.hjg.spring.ioc.injection.InjectionServiceImpl">
    <constructor-arg name="injectionDao" ref="injectionDao"></constructor-arg>
  </bean>
```

### 容器运行流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/3617e18fefa74a59adbc27529756ce24.png#pic_center)

## Bean ##
### Bean的定义 ###
- xml配置
	- 包括id, class, scope ,constructor arguments, properties, autowiring mode, lazy-initialization mode, initialization/destruction method。
	- 其中class是必须项。
	- bean的id大小写敏感。如果bean不配置id，默认id为：包名.类名#从0开始的自然数(如果为0，则#0可以省略)。
	- 如果bean只配置了name属性，但是没有配置id属性，默认id=name。
	```xml
	<bean id="beanAnnotation" class="com.hjg.spring.bean.annotation.componentScope.BeanAnnotation"></bean>
	```
-  注解
	- 可以自动发现并注册@Component, @Repository, @Service, @Controller注解的类，或者使用使用Component的自定义注解。
	- 开启使用注解的配置, context:component-scan：
	```xml
	 <context:component-scan base-package="com.hjg.spring.bean.annotation"/>
	```
	
	- @Componen等注解的值为bean的id，默认为首字母小写的类名
	```java
	@Component("beanAnnotation")
	public class BeanAnnotation
	```
	- 可以在name-generator属性中自定义命名策略，需要实现 **BeanNameGenerator** 接口。
	```xml
	<context:component-scan base-package="com.hjg.spring.bean.annotation"
	    name-generator="xxxxx"/>
	```
	- 同名的bean会覆盖，最后加载的生效

### 作用域 ###
- xml配置
	- 5种作用域：
	    1. singleton：单例，一个bean容器中只存在一个实例。
	    2. prototype：每次请求创建新的实例，此时 **destroy方法不生效** ，即对象资源的回收交由应用程序而不是spring框架完成。
	    3. request：每次http请求创建一个实例，且在当前request中有效。
	    4. session：每次http请求创建一个实例，且在当前session中有效。
	    5. global session：在基于portlet的web中有效，否则同session。
	- scope的默认模式为单例，singleton。
	```xml
	 <bean id="beanScope" class="com.hjg.spring.bean.scope.BeanScope" scope="prototype"></bean>
	```
- 注解
	- @Scope默认为singleton。
	```java
	@Scope("prototype")
	public class BeanAnnotation 
	```
	- 可以在scope-resolver属性中自定义scope策略，需要实现 **ScopeMetadataResolver** 接口。
	```xml
	<context:component-scan base-package="com.hjg.spring.bean.annotation" scope-resolver="xxxxx"/>
	```
### 生命周期
- **实例化**：newInstance分配内存空间
- **初始化**：
	- 填充属性。
    - 实现org.springframework.beans.factory.InitializingBean接口，覆盖afterPropertiesSet方法。
    - bean中配置init-method。
    - 各种Aware接口。
    - 实现BeanPostProcessor接口，调用 postProcessBeforeInitialzation和postProcessAfterInitialization方法。
      - [如监控bean启动时间](https://mp.weixin.qq.com/s/a_wfIh8roHrbG0_jXFT8Lw)
- **销毁**：
    - 实现org.springframework.beans.factory.DisposableBean接口，覆盖destroy方法。
    - bean中配置destroy-method。
    - 实现DisposableBean接口，调用destroy方法。
### 循环依赖问题
![在这里插入图片描述](https://img-blog.csdnimg.cn/7f97962e866a453bb221a46f3dee56f5.png)
- 使用多级缓存解决循环依赖 
- 使用三级缓存而不是二级缓存的原因：未完成的bean可能是简单对象或者aop增强前的对象，也可能是**aop增强**后的对象，需要区分。

### 自动装配 ###
#### xml配置 ####
- 默认不自动装配。
- byName：根据bean的id和属性名自动装配。
    - 事实上是通过bean的id寻找具有对应**方法名称**的set方法，使用该方法注入。
- byType：根据bean的class和属性类型自动装配。如果属性对应多个类型的bean，则抛出异常。
    - 事实上是通过bean的class寻找具有相同**参数类型**的set方法，使用该方法注入。
- constructor：类似byType，但是寻找构造器而不是set方法。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"
  default-autowire="byName">
</beans>
```

#### 注解 ####
-  @Autowired注解，可以使用在传统的setter方法、构造器、成员变量上。
-  如果没有合适的bean会抛出 NoSuchBeanDefinitionException 异常，可以设置required=false避免抛出异常，但是使用的使用可能会空指针。
```java
    @Autowired(required = false)
```
-  可以对集合参数使用@Autowired注解。
    - list放入bean的实例，map放入bean的id和实例。
    - 如果需要数组有序，可以使bean实现 Ordered 接口，或者使用@Order注解。排序只对类型list的数据结构有效，值越小优先级越高，相当于定义了一个**比较器**。 
```java
    @Autowired
    private List<BeanInterface> list;
    @Autowired
    private Map<String, BeanInterface> map;
```
- 自动装配可能出现多个bean实例时，可以用@Qualifier注解缩小范围。
```java
    @Autowired
    @Qualifier("beanImplOne")
    private BeanInterface beanInterface;
```

### Aware接口 ###
- 实现Aware接口的bean在初始化之后可以**获取并操作资源**。
    - ApplicationContextAware获取并操作applicationContext。
    - BeanNameAware获取并操作beanName。
```java
public class OneApplicationContext implements ApplicationContextAware, BeanNameAware {

    private String beanName;
    private ApplicationContext applicationContext;

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        //初始化bean之后的操作
        this.applicationContext = applicationContext;
        System.out.println("bean hashcode: " + applicationContext.getBean(beanName).hashCode());
    }


    public void setBeanName(String name) {
        this.beanName = name;
    }
}
```
### Resource接口 ###
- Resource：关于资源文件的统一接口，包括UrlResource, ClassPathResource, FileSystemResource。
- ResourceLoader：用于获取资源。**ApplicationContext实现** ResourceLoader接口。
    - ResourceLoader前缀包括：classpath, file, http, 无前缀(依赖于加载器，默认前缀为classpath)。
```java
    public void resource() throws IOException {
        Resource resource = applicationContext.getResource("classpath:jdbc.properties");
        Resource resource2 = applicationContext.getResource("file:A:\\eclipse\\workspace\\exercise\\ssm\\src\\main\\resources\\jdbc.properties");
        Resource resource3 = applicationContext.getResource("http://www.baidu.com");
    }
```

### @ Bean注解 ###
- @Bean可以标识一个用于配置和初始化由IOC容器管理的新对象的**方法**，类似<bean>标签，也可以自定义name、配置init和destroy方法。
- 通常在@Configuration标识的类中使用@Bean。
```java
@Configuration
public class StoreConfig {

    @Autowired
    private Store<String> s1;

    @Autowired
    private Store<Integer> s2;

    @Bean(name = "stringStore", initMethod = "init", destroyMethod = "destroy")
    @Scope(value = "prototype")
    public StringStore stringStore() {
        return new StringStore();
    }

    @Bean
    public IntegerStore integerStore() {
        return new IntegerStore();
    }
}
```
- 之前stringStore的定义相当于：
```xml
  <bean id="stringStore" name="stringStore" class="com.hjg.spring.bean.annotation.container.StringStore"
    init-method="init" destroy-method="destroy" scope="prototype">
  </bean>
```

### @ ImportResource注解 ###
- @ImportResource注解的值为spring xml文件路径，引入对应的配置文件。
```java
@Configuration
@ImportResource("classpath:config.xml")
public class ImportConfig
```
- 如果xml配置文件中引入properties文件，可以使用@Value和全局变量给类的属性赋值，这相当于配置property属性。
```xml
  <context:property-placeholder location="classpath:config.properties"/>
```
```java
    /**
     * 配置文件里需要写jdbc.username,如果是用${username}引入则为系统用户名
     */
    @Value("${jdbc.username}")
    private String JdbcUsername;
```
```xml
    <property name="JdbcUsername" value="${jdbc.username}"/>
```



## AOP ##
- **面向切面编程**（Aspect Oriented Programming, AOP），实现程序功能的统一维护。
    - 主要功能为：日志记录、性能统计、安全控制、事务处理、异常处理等。
    - 实现方式为：
        1. 预编译：AspectJ。
        2. 运行期动态代理(jdk动态代理、cglib动态代理)：SpringAOP、jbossAOP。
    - SpringAOP：
        - 为纯java实现，没有提供最完整的AOP实现，只支持方法执行**连接点**，这是为了侧重于提供一种AOP实现和IOC容器的**整合**。
        - Spring默认使用jdk动态代理实现AOP，也可以使用cglib代理。
- 几个概念。<br>![aopContent.png](https://img-blog.csdnimg.cn/img_convert/904b3ac0ed7d2e11bda57034de7e8935.png)
- 关键为**切面**,是通知和切点的结合。
    - **通知**：定义切面**做什么**，**何时使用**。
    - **切点**：定义切面在**何处使用**，会匹配通知所需要织入的一个或多个连接点。

### Advice ###
- 通知的类型：<br>![aopAdvice.png](https://img-blog.csdnimg.cn/img_convert/ae37d3e70784e2beb60fb7670060e6a6.png)
- spring xml的配置：
    - aop:aspect的ref为定义切面的bean的id。
    - aop:pointcut中的expression限定切面的影响范围。也可以在pointcut属性中定义影响范围。
    - 环绕通知需要在ProceedingJoinPoint实例执行proceed方法前后进行操作。可以使用 args 向通知提供切入点方法的参数。
    - aop:declare-parents为匹配到的类添加**父类接口**并提供这个接口的实现。
```xml
  <aop:config>
    <aop:aspect id="oneAspectAOP" ref="oneAspect">
      <aop:pointcut id="onePointcut"
        expression="execution(* com.hjg.spring.aop.advice.*Biz.*(..))"></aop:pointcut>
      
      <aop:before method="before" pointcut-ref="onePointcut"></aop:before>
      <aop:after-returning method="afterReturning" pointcut-ref="onePointcut"></aop:after-returning>
      <aop:after-throwing method="afterThrowing" pointcut-ref="onePointcut"></aop:after-throwing>
      <aop:after method="after" pointcut-ref="onePointcut"></aop:after>
      <aop:around method="around" pointcut-ref="onePointcut"></aop:around>

      <aop:around method="aroundInit"
        pointcut="execution(* com.hjg.spring.aop.advice.*Biz.init(String,int)) and args(bizName,times)"></aop:around>

      <aop:declare-parents types-matching="com.hjg.spring.aop.advice.*"
        implement-interface="com.hjg.spring.aop.introductions.Fit"
        default-impl="com.hjg.spring.aop.introductions.FitImpl"></aop:declare-parents>
    </aop:aspect>
  </aop:config>
```
```java
    public Object aroundInit(ProceedingJoinPoint pjp, String bizName, int times) throws Throwable {
        System.out.println("OneAspect aroundInit 1");
        Object obj = pjp.proceed();
        System.out.println("OneAspect aroundInit 2");
        return obj;
    }
```
```java
    public void init(String bizName, int time) {
        this.bizName = bizName;
        this.time = time;
    }
```

### Advisor ###
- 只有一个advice的切面。

### AspectJ ###
- 使用**注解**方式实现AOP。
- 切面使用@Aspect, @Component标识。
```java
@Component
@Aspect
public class OneAspect 
```
- 切入点通过返回值void的方法定义，使用@Pointcut注解。
```java
    @Pointcut("execution(* com.hjg.spring.aop.aspectj.biz.*Biz.*(..))")
    public void pointcut() {
    }
```
- 通知可以使用注解，@Before、@AfterReturning、@AfterThrowing、@After、@Around标识。
    - args传递切入点方法的参数，annotation传递切入点方法的注解。
    - returning传递返回值，throwing传递抛出的异常。
```java
 @Before("pointcut() && args(arg) && @annotation(oneMethod)")
 public void before(String arg, OneMethod oneMethod){}

 @AfterReturning(pointcut = "bizPointcut()", returning = "returnValue")
 public void afterRuturning(Object returnValue){}

 @AfterThrowing(pointcut = "bizPointcut()", throwing = "e")
 public void afterThrowing(RuntimeException e){}
```
```java
    @OneMethod("one method")
    public String save(String arg) {
        System.out.println("OneBiz save : " + arg);
        return " Save success!";
    }
```
- @DeclareParents注解为配置中的aop:declare-parents，为匹配到的类添加**父类接口**并提供这个接口的实现。

## 事务 ##
### 概念 ###
- 一组数据库操作，要么全部执行，要么全部不执行，具有ACID性。
- [http://blog.csdn.net/qq_40369829/article/details/78477023#事务](http://blog.csdn.net/qq_40369829/article/details/78477023#事务)

### API ###
- 抽象为3个接口。

#### 事务管理器 ####
- PlatformTransactionManager<br>![PlatformTransactionManager.png](https://img-blog.csdnimg.cn/img_convert/9bf3fe72bbb207a6ae03a535cf4a69eb.png)
- 为不同持久层框架提供了不同的事务管理器实现。<br>![TransactionManagerWithDao.png](https://img-blog.csdnimg.cn/img_convert/d85fd59de2a4a1d377e6808052b2cad5.png)

#### 事务定义信息 ####
- TransactionDefinition<br>![TransactionDefinition.png](https://img-blog.csdnimg.cn/img_convert/08c5810c9249b044c13a897b26cac359.png)
- 事务的隔离级别可以不同程度上解决，**脏读**、**不可重复读**、**虚读**（幻读）的问题。mysql默认为REPEATABLE_READ级别。<br>![isolation.png](https://img-blog.csdnimg.cn/img_convert/7300d91a87f9788b1de70bad08a16c5f.png)
- 事务的传播行为可以分为3类，共7种：在一个事务中，不在一个事务中，嵌套事务。<br>![TransactionTransport.png](https://img-blog.csdnimg.cn/img_convert/a18b536486cf59dbcdbdfc9eae56b8ac.png)

#### 事务状态 ####
- TransactionStatus<br>![TransactionStatus.png](https://img-blog.csdnimg.cn/img_convert/d82d3633d57613bbbbd2e595c11bd26b.png)
 
### 事务管理 ###
- 编程式的事务管理。
    - 通过事务模板 TransactionTemplate 手动配置，使用少。 
- 使用XML配置的声明式事务管理，使用AOP实现。
    - 代理配置：需要为每个事务管理的类配置代理增强，使用少。
    - xml配置：配置好之后，类上无需添加代码，经常使用。
    - 注解配置：使用 @Transactional 注解,配置简单，经常使用。

#### 编程式事务管理 ####
```java
    public void transfer(final String out, final String in, final Double money) {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                accountDao.outMoney(out, money);
                accountDao.inMoney(in, money);
            }
        });
    }
```

#### 声明式事务管理 ####
> 3种配置方式。

- 代理配置。
    - TransactionProxyFactoryBean中注入事务属性。
```xml
  <!--引入属性文件-->
  <context:property-placeholder location="classpath:jdbc.properties"/>

  <!--c3p0连接池-->
  <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
    <property name="driverClass" value="${jdbc.driver}"/>
    <property name="jdbcUrl" value="${jdbc.url}"/>
    <property name="user" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
  </bean>

  <!--配置Service-->
  <bean id="accountService4" class="com.hjg.spring.transaction.demo4.AccountServiceImpl">
    <property name="accountDao" ref="accountDao4"/>
  </bean>

  <!--配置DAO-->
  <bean id="accountDao4" class="com.hjg.spring.transaction.demo4.AccountDaoImpl">
    <property name="dataSource" ref="dataSource"/>
  </bean>

  <!--配置事务管理器-->
  <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
  </bean>

  <!--配置业务层代理-->
  <bean id="accountServiceProxy" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <!--配置目标对象-->
    <property name="target" ref="accountService"/>
    <!--注入事务管理器-->
    <property name="transactionManager" ref="transactionManager"/>
    <!--注入事务属性-->
    <property name="transactionAttributes">
      <props>
        <!--
          PROPAGATION :事务的传播行为
          ISOLATION :事务的隔离级别
          readonly :只读
          -Exception :发生哪些异常回滚事务
          +Exception :发生哪些异常不回滚事务
        -->
        <prop key="transfer">PROPAGATION_REQUIRED,+java.lang.ArithmeticException</prop>
      </props>
    </property>
  </bean>
```
- xml配置。
    - 配置事务通知。
    - 配置切面。
```xml
  <!--直到配置事务管理器同上-->
  <!--配置事务通知，事物的增强-->
  <tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
      <!--
      propagation,isolation,read-only,rollback-for,no-rollback-for,timeout
      -->
      <tx:method name="transfer" propagation="REQUIRED" isolation="DEFAULT"/>
    </tx:attributes>
  </tx:advice>

  <!--配置切面-->
  <aop:config>
    <!--切入点-->
    <aop:pointcut id="pointcut1"
      expression="execution(* com.hjg.spring.transaction.demo3.AccountService+.*(..))"></aop:pointcut>
    <!--切面-->
    <aop:advisor advice-ref="txAdvice" pointcut-ref="pointcut1"/>
  </aop:config>
```
- 注解配置。
    - 开启事务注解。
    - 在需要使用事务的类或者方法上使用 @Transactional 注解。
```xml
  <!--直到配置事务管理器同上-->
  <!--开启注解事务-->
  <tx:annotation-driven transaction-manager="transactionManager"/>
```
```java
@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.DEFAULT, readOnly = false)
```

## 参考 ##
- [为了忽悠大厂面试官，熬夜总结了这些Spring面试题](https://mp.weixin.qq.com/s/9H8yX2CUSbFoE-6YshUbbA)
- [深入理解 Spring finishBeanFactoryInitialization](https://www.cnblogs.com/ZhuChangwu/p/11755973.html)