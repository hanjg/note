[toc]
## 测试 ##
- 自动生成的Test类中，@RunWith(SpringRunner.class) 开启Spring集成测试。@SpringBootTest 自动搜索应用启动类，加载应用的上下文。
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ReadinglistApplicationTests
```

- 针对 **Controller测试**，可以忽略RequestMapping等标签，直接测试方法本身，但是无法处理post请求，也无法测试表单和参数的绑定。spring boot有两种方法可以模拟http请求测试：
  1. spring mock mvc：在模拟servlet容器里测试控制器。启动快。
  2. web继承测试：在嵌入式servlet容器中启动应用测试。接近真实环境。

### 模拟spring mvc ###
- 获取web应用上下文，并注入模拟的mvc。
```java
    @Autowired
    private WebApplicationContext webApplicationContext;
    private MockMvc mockMvc;

    @Before
    public void setupMockMvc() {
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
    }
```

- 编排测试逻辑。需要使用模拟请求测试和校验类MockMvcRequestBuilders，MockMvcResultMatchers，Matchers。
```java
 package com.example.readinglist;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.hamcrest.Matchers.*;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.context.web.WebAppConfiguration;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

@RunWith(SpringRunner.class)
@SpringBootTest
@WebAppConfiguration
public class ReadinglistApplicationTests {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private WebApplicationContext webApplicationContext;
    private MockMvc mockMvc;

    @Before
    public void setupMockMvc() {
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
    }

    @Test
    public void testBook() throws Exception {
        Book book = getBook();
        String read1Url = "/readingList/reader1";

        //查询图书，结果为空
        mockMvc.perform(get(read1Url))
                .andExpect(status().isOk())
                .andExpect(view().name("readingList"))
                .andExpect(model().attributeExists("books"))
                .andExpect(model().attribute("books", is(empty())));
        //添加图书
        mockMvc.perform(post(read1Url)
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .param("title", book.getTitle())
                .param("author", book.getAuthor())
                .param("isbn", book.getIsbn())
                .param("description", book.getDescription()))
                .andExpect(status().is3xxRedirection())
                .andExpect(header().string("Location", read1Url));
        //查询图书，结果为新加的图书
        mockMvc.perform(get(read1Url))
                .andExpect(status().isOk())
                .andExpect(view().name("readingList"))
                .andExpect(model().attributeExists("books"))
                .andExpect(model().attribute("books", hasSize(1)))
                .andExpect(model().attribute("books", contains(samePropertyValuesAs(book))));
    }

    private Book getBook() {
        Book book = new Book();
        book.setReader("reader1");
        book.setId(1L);
        book.setTitle("title1");
        book.setAuthor("author1");
        book.setIsbn("12345");
        book.setDescription("desc");
        return book;
    }
}

```

### 服务器测试 ###
- springboot1.5之前可以使用WebIntegrationTest注解，启动嵌入式的servlet容器。[参考](https://www.cnblogs.com/alliswelltome/p/5948973.html)。
- 同时可以使用Selenium框架进行页面测试。
- spring1.5之后暂未找到替代方案。

## 部署 ##
- 部署可以直接jar包部署，也可以使用内置的tomcat和外部的tomcat。[参考](https://blog.csdn.net/fanshukui/article/details/80258793)。
### jar部署 ###
- nohup + & 后台运行任务。
```sh
nohup java -jar house.jar &
exit
```

### 内置tomcat ###
- 在application.yml中微调默认配置，如：
```yml
server:
  port: 8080
  servlet:
    context-path: /house
```

- 打包成jar之后用命令行运行。

### 外部tomcat ###
- 修改启动类，继承SpringBootServletInitializer实现configure方法，相当于之前的web.xml，加载spring上下文。
```java
@SpringBootApplication
@EnableTransactionManagement
public class HouseApplication extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(HouseApplication.class, args);
    }

    /**
     * 与在web.xml中配置负责初始化Spring应用上下文的监听器作用类似
     */
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(HouseApplication.class);
    }
}
```

- 由于tomcat有外部容器提供，需要将依赖设为provided，以防与内置容器冲突。
```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.tomcat.embed</groupId>
      <artifactId>tomcat-embed-jasper</artifactId>
      <!--springboot自带的tomcat并没有携带tomcat-embed-jasper的依赖。
      使用内置tomcat的时候需要设为默认的complie-->
      <scope>provided</scope>
    </dependency>
```

- 打包为war之后放入tomcat的webapps启动。
