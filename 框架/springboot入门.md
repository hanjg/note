


@[toc]
## spring boot 核心 ##
- 其设计目的是用来简化新spring应用的初始搭建以及开发过程。
- 从本质上说，spring boot就是spring，只是利用了spring4的**条件化配置特性**和maven或者gradle提供的**传递依赖解析**，实现spring应用程序上下文里的自动配置。
- [spring基础](https://blog.csdn.net/qq_40369829/article/details/79355424)，[spring mvc基础](https://blog.csdn.net/qq_40369829/article/details/79355749)
- 有以下4个核心特征。

### 自动配置 ###
- 在程序启动时会根据spring-boot-autoconfigure.jar中的配置类自动配置。由于这些配置类使用**条件化配置**，只有满足一定条件才会生效。
- 比如DataSourceAutoConfiguration类，由于ConditionalOnClass注解，需要classpath里面有DataSource.class, EmbeddedDatabaseType.class，才能配置，否则被忽略。
```java
@Configuration
@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})
@EnableConfigurationProperties({DataSourceProperties.class})
@Import({DataSourcePoolMetadataProvidersConfiguration.class, DataSourceInitializationConfiguration.class})
public class DataSourceAutoConfiguration {
}
```

### 起步依赖 ###
- 利用传递依赖解析，把常用的库聚合，组成为特定功能定制的依赖。
    - 如spring-boot-starter-web，依赖spring-core,spring-web,spring-mvc等。
- 如果需要排除传递依赖，可以使用exclusions。
```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <exclusions>
        <exclusion>
          <groupId>com.fasterxml.jackson.core</groupId>
        </exclusion>
      </exclusions>
    </dependency>
```

- 可以覆盖传递依赖，例如重新指定版本号。maven总会使用最近加载的依赖，gradle使用库的最新版本。

### 命令行界面 ###
- spring boot CLI（命令行接口），只需代码就能完成完整的应用程序，无须传统项目构建。
- spring的非必要组成部分，是一套不太常规的开发模型。

### Actuator ###
- 监视运行中的spring应用内部情况。
    - 如上下文中的bean，环境变量，线程状态，内存用量等等各种指标。

## IDEA创建第一个spring boot应用 ##
### 搭建框架 ###
- 在new project中使用spring initializr。<br>![181202.initializr.png](https://img-blog.csdnimg.cn/20181202223247121.png)
- 配置maven元数据<br>![181202.metadata.png](https://img-blog.csdnimg.cn/20181202223450875.png)
- 选择需要的功能。<br>![181202.dep.png](https://img-blog.csdnimg.cn/20181202223747596.png)
- 创建项目，自动生成如下结构。<br>![181202.struct.png](https://img-blog.csdnimg.cn/20181202225033465.png)

### 目录结构 ###
- ReadinglistApplication配置和启动引导，SpringBootApplication注解相当于三个注解：Configuration,ComponentScan,EnableAutoConfiguration
```java
@SpringBootApplication
public class ReadinglistApplication {

    public static void main(String[] args) {
        SpringApplication.run(ReadinglistApplication.class, args);
    }
}
```

- application.properties配置应用属性，如服务端口server.port = 8080。

- static目录存放静态资源，如css。
- templates目录存放视图模板。
- ReadinglistApplicationTests是一个测试类骨架，其中自动生成的空方法可以检查上下文加载是否正常。
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = ReadinglistApplication.class)
public class ReadinglistApplicationTests {

    @Test
    public void contextLoads() {
    }

}
```

- pom.xml指定了spring-boot的版本号，起步依赖，使用的spring插件。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.example</groupId>
	<artifactId>readinglist</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>readinglist</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

### 开发功能 ###
- 增加模型。
```java
package com.example.readinglist;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

/**
 * @Author: hjg
 * @Date: Create in 2018/12/2 18:32
 * @Description:
 */
@Entity
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String reader;
    private String isbn;
    private String title;
    private String author;
    private String description;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getReader() {
        return reader;
    }

    public void setReader(String reader) {
        this.reader = reader;
    }

    public String getIsbn() {
        return isbn;
    }

    public void setIsbn(String isbn) {
        this.isbn = isbn;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getAuthor() {
        return author;
    }

    public void setAuthor(String author) {
        this.author = author;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }
}

```

- 继承JapRepository接口（包含常用dao方法），并扩展接口。spring data中，只需要定义仓库接口，接口会在启动时自动实现。
```java
package com.example.readinglist;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * @Author: hjg
 * @Date: Create in 2018/12/2 18:35
 * @Description:
 */
public interface ReadingListRepository extends JpaRepository<Book, Long> {

    List<Book> findByReader(String reader);

}
```

- 创建controller。
```java
package com.example.readinglist;

import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

/**
 * @Author: hjg
 * @Date: Create in 2018/12/2 18:36
 * @Description:
 */
@Controller
@RequestMapping("/readingList")
public class ReadingListController {

    private ReadingListRepository readingListRepository;

    @Autowired
    public ReadingListController(ReadingListRepository readingListRepository) {
        this.readingListRepository = readingListRepository;
    }

    @RequestMapping(value = "/{reader}", method = RequestMethod.GET)
    public String readersBooks(@PathVariable String reader, Model model) {
        List<Book> readingList = readingListRepository.findByReader(reader);
        if (readingList != null) {
            model.addAttribute("books", readingList);
        }
        return "readingList";
    }

    @RequestMapping(value = "/{reader}", method = RequestMethod.POST)
    public String addToReadingList(@PathVariable String reader, Book book) {
        book.setReader(reader);
        readingListRepository.save(book);
        return "redirect:/readingList/{reader}";
    }
}
```

- 增加静态资源。static下面增加style.css，templates下面增加readingList.html。
```css
body {
	background-color: #cccccc;
	font-family: arial,helvetica,sans-serif;
}

.bookHeadline {
	font-size: 12pt;
	font-weight: bold;
}

.bookDescription {
	font-size: 10pt;
}

label {
	font-weight: bold;
}
```
```html
<html>
  <head>
    <title>Reading List</title>
    <link rel="stylesheet" th:href="@{/style.css}"></link>
  </head>

  <body>
    <h2>Your Reading List</h2>
    <div th:unless="${#lists.isEmpty(books)}">
      <dl th:each="book : ${books}">
        <dt class="bookHeadline">
          <span th:text="${book.title}">Title</span> by 
          <span th:text="${book.author}">Author</span>
          (ISBN: <span th:text="${book.isbn}">ISBN</span>)
        </dt>
        <dd class="bookDescription">
          <span th:if="${book.description}" 
                th:text="${book.description}">Description</span>
          <span th:if="${book.description eq null}">
                No description available</span>
        </dd>
      </dl>
    </div>
    <div th:if="${#lists.isEmpty(books)}">
      <p>You have no books in your book list</p>
    </div>

    
    <hr/>
    
    <h3>Add a book</h3>
    <form method="POST">
      <label for="title">Title:</label>
        <input type="text" name="title" size="50"></input><br/>
      <label for="author">Author:</label>
        <input type="text" name="author" size="50"></input><br/>
      <label for="isbn">ISBN:</label>
        <input type="text" name="isbn" size="15"></input><br/>
      <label for="description">Description:</label><br/>
        <textarea name="description" cols="80" rows="5"></textarea><br/>
      <input type="submit"></input>
    </form>
    
  </body>
</html>
```

### 运行 ###
- 直接运行启动类(也可以打包成jar或者war)。<br>![181202.run.png](https://img-blog.csdnimg.cn/20181202231316678.png)
- 访问localhost:8080/readingList/reader1<br>![181202.web.png](https://img-blog.csdnimg.cn/20181202231720934.png)

## 实现原理
### 启动流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/0bb5651b8c8d4c0cab7638ac1e49eafe.png)
- springboot在spring基础上通过**BeanFactoryPostProcessor**扩展自动装配。[参考](https://www.cnblogs.com/trgl/p/7353782.html)。
### 自动装配
1. 启动类的 @SpringBootApplication注解继承@EnableAutoConfiguration，改注解在@Import注解中引入AutoConfigurationImportSelector.class
2. 容器refresh时，在invokeBeanFactoryPostProcessors底层读取启动类的@Import注解，读取的位置：org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getAnnotationClass
3. AutoConfigurationImportSelector实现ImportSelector接口的selectImports方法，在```META-INF/spring.factories```下读取org.springframework.boot.autoconfigure.EnableAutoConfiguration的实现类。
4. 这些实现类包含@Configuration注解，返回该类中@Bean注解声明的bean

## spring boot参考资料 ##
- [springboot系列](https://blog.csdn.net/Winter_chen001/article/details/80537829)
- [springboot样例](https://github.com/ityouknow/spring-boot-examples)