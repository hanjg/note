[toc]
## 特点 ##
- 是构建Web应用程序的全功能MVC模块。
- 提供了一个DispatcherServlet作用前端控制器来分派请求，同时提供灵活的配置处理程序映射、视图解析、语言环境和主题解析，并支持文件上传。
- 还包含了多种视图技术，例如JSP、Velocity、Tiles、iText和POI等。
- 分离了控制器、模型对象、分派器以及处理程序对象的角色，这种分离让它们更容易进行定制。
- 核心思想：业务数据**抽取**和业务数据**呈现**分离。
- [流程](http://blog.csdn.net/qq_40369829/article/details/78477116#spring-mvc)


## 基本配置 ##
- 基本配置和开发可参考：[http://blog.csdn.net/qq_40369829/article/details/79323270#web层](http://blog.csdn.net/qq_40369829/article/details/79323270#web层),以及如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:context="http://www.springframework.org/schema/context"
  xmlns:aop="http://www.springframework.org/schema/aop"
  xmlns:tx="http://www.springframework.org/schema/tx"
  xmlns:mvc="http://www.springframework.org/schema/mvc"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">
  <!--激活注释，如@Required,@Autowired-->
  <context:annotation-config/>

  <!--DispatcherServlet上下文，只管理@Controller标注的bean, 忽略其他bean, 如@Service-->
  <context:component-scan base-package="com.hjg.springmvc">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
  </context:component-scan>

  <!--HandlerMapping无需配置，Spring MVC默认启动-->

  <!--扩充注解驱动，将请求参数绑定到控制器参数-->
  <mvc:annotation-driven/>

  <!--静态资源,css,img等-->
  <mvc:resources mapping="/resources/**" location="/resources/"/>

  <!--ViewResoler可以配置多个，按照order属性排序。但是InternalResourceViewResolver需要放在最后-->
  <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value="/WEB-INF/jsps/"/>
    <property name="suffix" value=".jsp"/>
  </bean>
</beans>
```

## 文件上传 ##
- mvc的xml中配置CommonsMultipartResovler。
```xml
  <!--解析上载文件.懒加载可以推迟文件解析，以捕获文件大小异常-->
  <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <property name="maxUploadSize" value="209715200"/>
    <property name="defaultEncoding" value="UTF-8"/>
    <property name="resolveLazily" value="true"/>
  </bean>
```
- Controller需要使用MultipartFile类接收文件，包括上传文件和接收上传文件的操作。
```java
    /**
     * 上传文件页面
     */
    @RequestMapping(value = "/upload", method = RequestMethod.GET)
    public String showUploadPage() {
        return "file";
    }

    /**
     * 接收上传文件
     */
    @RequestMapping(value = "/doUpload", method = RequestMethod.POST)
    public String doUploadFile(@RequestParam("file") MultipartFile file) throws IOException {
        if (!file.isEmpty()) {
            FileUtils.copyInputStreamToFile(file.getInputStream(),
                    new File("C:\\temp\\", System.currentTimeMillis() + file.getOriginalFilename()));
        }
        return "success";
    }
```
- 使用表单的简单上传页面。
```html
<div align="center">
    <h1>上传附件</h1>
    <form method="post" action="/doUpload" enctype="multipart/form-data">
        <input type="file" name="file"/>
        <input type="submit"/>
    </form>
</div>
```

## 处理JSON ##
- mvc的xml中配置ContentNegotiatingViewResolver，可以用同样的数据呈现不同的view。
```
  <bean
    class="org.springframework.web.servlet.view.ContentNegotiatingViewResolver">
    <property name="order" value="1"/>
    <property name="mediaTypes">
      <map>
        <entry key="json" value="application/json"/>
        <entry key="xml" value="application/xml"/>
        <entry key="htm" value="text/html"/>
      </map>
    </property>
    <property name="defaultViews">
      <list>
        <!-- JSON View -->
        <bean
          class="org.springframework.web.servlet.view.json.MappingJackson2JsonView">
        </bean>
      </list>
    </property>
    <property name="ignoreAcceptHeader" value="true"/>
  </bean>
```
- Controller中使用@ResponseBody返回json。
```java
    @RequestMapping(value = "viewInJson/{courseId}", method = RequestMethod.GET)
    public @ResponseBody
    Course getCourseInJson(@PathVariable Integer courseId) {
        return courseService.getCoursebyId(courseId);
    }
```

## 过滤器和拦截器 ##
### 过滤器 ###
- web.xml中配置CharacterEncodingFilter，将请求字符转为utf-8.
```xml
  <filter>
    <filter-name>encoding</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>utf8</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>encoding</filter-name>
    <!--url-pattern不能为'/'-->
    <url-pattern>*</url-pattern>
  </filter-mapping>
```

### 拦截器 ###
- 拦截器统一拦截从浏览器发往**控制器（ Controller ）** 的请求来完成功能的**增强**。

#### 使用 ####
- 编写拦截器类，实现HandlerInterceptor接口。
```java
public class TestIntercepter1 implements HandlerInterceptor {
    /**
     * @param handler 被拦截的请求对象，controller
     * @return true:请求继续执行；false:请求终止，不会到达控制器。
     */
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        request.setCharacterEncoding("utf8");
        //其他操作
        return true;
    }

    /**
     * @param modelAndView 改变视图名称，或者发往视图的参数
     */
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
            ModelAndView modelAndView) throws Exception {
        modelAndView.addObject("msg", "msg from intercepter");
        modelAndView.setViewName("success3");
    }

    /**
     * 资源的销毁，如流。不常用。
     */
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
    }
}
```
- 在mvc的xml中配置拦截器。
```xml
  <mvc:interceptors>
    <mvc:interceptor>
      <mvc:mapping path="/viewAll"/>
      <bean class="com.hjg.springmvc.intercepter.TestIntercepter1"/>
    </mvc:interceptor>
  </mvc:interceptors>
```

#### 多个拦截器 ####
- 拦截器的顺序和mvc的xml中配置顺序一致。
- 拦截器的执行顺序，类似栈。<br>![multiIntercepter.png](http://img.blog.csdn.net/20180219111618544)

### 两者区别 ###
- 过滤器依赖于 **servlet容器**，基于回调函数，过滤范围大。
- 拦截器依赖于**框架容器**，基于反射机制，只过滤请求。
