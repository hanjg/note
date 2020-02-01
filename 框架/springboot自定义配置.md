[toc]
## 覆盖自动配置 ##
- spring boot加载应用级配置，之后在考虑自动配置类。 **ConditionalOnMissingBean** 注解是覆盖自动配置的关键，自定义配置之后将不再自动配置。
    - 例如SpringBootWebSecurityConfiguration只有在用户未自定义WebSecurityConfigurerAdapter的实现类时才会自动配置。
```java
@Configuration
@ConditionalOnClass({WebSecurityConfigurerAdapter.class})
@ConditionalOnMissingBean({WebSecurityConfigurerAdapter.class})
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
public class SpringBootWebSecurityConfiguration
```

### 自定义安全配置 ###
- pom中增加安全依赖。
```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
```

- 继承WebSecurityConfigurerAdapter，使用显式安全配置覆盖自动配置，使得只有角色为READER的用户可以访问。
    - springboot2中[JpaRepository的findOne返回值类型](https://www.cnblogs.com/magotzis/p/8259998.html)为Optional，[optional用法](http://www.ibloger.net/article/3209.html)。
    - 如不重载configure(HttpSecurity http)，会跳转到自带的登录页面。<br>![](https://img-blog.csdnimg.cn/20181204204655888.png)
    - 如果不重载configure(AuthenticationManagerBuilder auth)方法，则意味着拦截请求之后发现不存在该用户，即无法通过认证。
    - Reader继承UserDetails，实现getAuthorities方法为求方便，将角色写为READER。
```java
package com.example.readinglist;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.Example;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;

/**
 * @Author: hjg
 * @Date: Create in 2018/12/4 17:20
 * @Description:
 */
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private ReaderRespository readerRespository;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/").access("hasRole('READER')")
                .antMatchers("/**").permitAll();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(new UserDetailsService() {
            @Override
            public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
                Reader reader = new Reader();
                reader.setUsername(username);
                logger.info("username: {}", username);
                return readerRespository.findOne(Example.of(reader)).orElse(null);
            }
        });
    }
}

package com.example.readinglist;

import java.util.Arrays;
import java.util.Collection;
import javax.persistence.Entity;
import javax.persistence.Id;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

/**
 * @Author: hjg
 * @Date: Create in 2018/12/4 17:31
 * @Description:
 */
@Entity
public class Reader implements UserDetails {

    @Id
    private String username;
    private String fullname;
    private String password;

    @Override
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    @Override
    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getFullname() {
        return fullname;
    }

    public void setFullname(String fullname) {
        this.fullname = fullname;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return Arrays.asList(new SimpleGrantedAuthority("READER"));
    }


    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}

package com.example.readinglist;

import org.springframework.data.jpa.repository.JpaRepository;

/**
 * @Author: hjg
 * @Date: Create in 2018/12/4 17:24
 * @Description:
 */
public interface ReaderRespository extends JpaRepository<Reader, String> {

}
```

## 通过属性文件配置 ##
- spring boot可以从多种属性源获得属性。优先级从高到低为：
    - 命令行参数
    - java:comp/env里的JNDI属性
    - JVN系统属性
    - 操作系统环境变量
    - 随机的带random.*前缀属性
    - 应用程序外的application.properties或者application.yml文件
    - 打包在应用程序里的application.properties或者application.yml文件
    - 使用PropertySource标注的属性
    - 默认属性
- application文件在config子目录的优先级高。

### 微调自动配置 ###
- 可以在application文件或者环境变量中覆盖某些属性，如：
- 禁用thymeleaf模板缓存，可以方便的显示修改之后的模板。配置spring.thymeleaf.cache=false，或者环境变量中spring_thymeleaf_cache=false。
- 配置嵌入式服务器。server.port=8443。
- 配置日志，默认logback。可以在classpath根目录创建logback.xml配置，也可以设置属性
```properties
logging.path=./logs
logging.file=xx.log
logging.level.root=WARN
logging.level.root.org.springframework.security=DEBUG
```
- 配置数据源，例如为开发和测试环境各配置一套。
```properties
spring.datasource.url=jdbc:mysql://localhost/readingList
spring.datasource.username=root
spring.datasource.password=root
```

### bean属性外置 ###
- 在application.properties或者在amazon.properties中添加属性。参考 [properties中添加属性](https://www.cnblogs.com/yinfengjiujian/p/8784975.html)。
```properties
amazon.associatedId=asid
```

- 用一个bean收集amazon开头的属性。
```java
@Component
//PropertySource指定属性的来源，如不配置默认从application.properties中读取。
@PropertySource("classpath:amazon.properties")
@ConfigurationProperties(prefix = "amazon")
public class AmazonProperties {

    private String associatedId;

    public String getAssociatedId() {
        return associatedId;
    }

    public void setAssociatedId(String associatedId) {
        this.associatedId = associatedId;
    }
}
```

- 将这个属性注入controller并写入模型，在页面上显示，注意thymeleaf需要使用 **th:** 来加载变量。<br>![181204.outbean.png](https://img-blog.csdnimg.cn/20181204233302831.png)
```java
 model.addAttribute("associatedId", amazonProperties.getAssociatedId());
```
```html
 <input type="text" name="associatedId" size="15" th:value="${associatedId}"></input><br/>
```

### 基于Profile配置属性文件 ###
- 通过application.properties中的spring.profiles.active属性激活Profile。
- 不同的Profile属性命名为： application-{profile}.properties，如 application-dev.properties。
- spring boot会根据profile加载对应的属性文件。

## 自定义错误页面 ##
- spring boot由**视图解析器**查找id为 **error** 的视图，如使用Thymeleaf，则错误页面为error.html。
- 自定义错误页面则是将自定的error.html放在templates文件夹下面。