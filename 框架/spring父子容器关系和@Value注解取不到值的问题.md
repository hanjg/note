[toc]
## 父子容器 ##
- Spring中可以包含多个容器，以SpringMVC为例， **Spring为父容器** ， **SpringMVC为子容器** 。
- 父容器中的bean对子容器的bean是可见的，但是子容器的bean对父容器的bean是不可见的。
- 父容器仅仅是 **bean（对象）** 对子容器可见，加载的**全局变量仍然相互独立**。
- 父子容器的关系类似于java的内部类。

## @Value注解 ##
- Spring中的 @Value 注解可以获得Spring容器中加载的全局变量。以注解的形式赋予bean的成员变量。
```java
@Value("${xxx}")
private String a;
```

## @Value注解取不到值 ##
### 环境 ###
- 一个mvc工程，配置文件如下：<br>![1804.struct.png](https://img-blog.csdn.net/20180420200358769)
- spring-service.xml中加载属性文件，扫描service包中的bean。
```xml
  <!-- 加载配置文件 -->
  <context:property-placeholder location="classpath:resource/*.properties"/>

  <!-- 扫描包加载Service实现类 -->
  <context:component-scan base-package="com.taotao.portal.service"/>
```

- springmvc-config.xml中扫描controller，配置视图解析器、拦截器。
```xml
  <context:component-scan base-package="com.taotao.portal.controller"/>
  <mvc:annotation-driven/>

  <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
  </bean>

  <!-- 拦截器配置 -->
  <mvc:interceptors>
    <mvc:interceptor>
      <!-- 拦截订单类请求 -->
      <mvc:mapping path="/item/**"/>
      <bean class="com.taotao.portal.interceptor.LoginInterceptor"/>
    </mvc:interceptor>
  </mvc:interceptors>
```

- web.xml中从spring-service.xml中加载bean到spring容器，从spring-config.xml中加载bean到springmvc容器，他们是父子关系。 
```xml
  <!-- 加载spring容器 -->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring/spring-*.xml</param-value>
  </context-param>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>

  <!-- springmvc的前端控制器 -->
  <servlet>
    <servlet-name>taotao-portal</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springmvc/springmvc-config.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>taotao-portal</servlet-name>
    <!--伪静态化，搜索引擎优化-->
    <url-pattern>*.html</url-pattern>
  </servlet-mapping>
```

- 拦截器实现类LoginInterceptor。自动组装service类，使用Value注解获取属性文件中定义的属性。
```java
public class LoginInterceptor implements HandlerInterceptor {

    private static final Logger LOGGER = LoggerFactory.getLogger(LoginInterceptor.class);

    @Autowired
    private UserService userService;

    @Value("${TT_TOKEN}")
    private String TT_TOKEN;
    @Value("${SSO_BASE_URL}")
    private String SSO_BASE_URL;
    @Value("${SSO_PAGE_LOGIN}")
    private String SSO_PAGE_LOGIN;


    /**
     * @return 是否放行
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object o)
            throws Exception {
        //从cookie中取token，根据token取得用户信息
        String token = CookieUtils.getCookieValue(request, "TT_TOKEN");
        TbUser user = userService.getUser(token);

        //取不到用户信息
        if (user == null) {
            //跳转到登录页面，把用户请求的url作为参数传递给登录页面。
            String redirectUrl = SSO_BASE_URL + SSO_PAGE_LOGIN
                    + "?redirect=" + request.getRequestURL();
            LOGGER.debug("redirect: {}", redirectUrl);
            response.sendRedirect(redirectUrl);
            return false;
        }

        return true;

    }

    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o,
            ModelAndView modelAndView) throws Exception {
    }

    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse,
            Object o, Exception e) throws Exception {
    }
}
```

### 现象 ###
- 访问拦截器拦截的url:/item/**的时候，跳转到预期之外的url，出现404。
- 查看日志，发现容器将 @Value注解中的值赋予变量SSO_BASE_URL、SSO_PAGE_LOGIN，并没有加载属性文件中的内容，从而未能实现应有的跳转。<br>![1804.interceptvalue.png](https://img-blog.csdn.net/2018042020182914)

### 原因 ###
- LoginInterceptor在spring-config.xml中配置，被加载到springMVC这一**子容器**中，属性文件的扫描在spring-service.xml中配置，被加载到spring这一**父容器**中。
- 父容器中的**bean(对象)**对LoginInterceptor是可见的，但是父容器加载的**属性变量**只是对父容器spring本身中的对象可见，所以LoginInterceptor使用 @Value注解无法在子容器springMVC中获得对应的属性值，只能取得容器赋予的默认值。

### 解决 ###
#### 再次加载 ####
- 在springMVC中再次扫描属性文件，加载到springMVC这一子容器中。
```xml
  <!-- 加载配置文件 -->
  <context:property-placeholder location="classpath:resource/*.properties"/>
```

#### bean传值 ####
- 在userService类中使用 @Value注解，加载spring的属性。因为service类对于子容器可见，可以传值。