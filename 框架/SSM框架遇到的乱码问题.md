[toc]
## 原因 ##
- 浏览器以某种编码发送请求（通常**UTF-8**），服务器接收请求，按照服务器的编码方式解码（tomcat默认**ISO8859-1**）。
- 这样**前后端编码方式不一致**导致乱码问题。

## 解决 ##
- tomcat中get和post处理不一样，乱码问题处理也不一样。
### post ###
- web.xml中配置过滤器。
```xml
  <filter>
    <filter-name>encoding</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>utf-8</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>encoding</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```

### get ###
- conf/server.xml中配置tomcat端口的编码，添加URIEncoding，比如修改8080端口的编码。
```java
<Connector port="8080"  protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" URIEncoding="UTF-8" />
```

- 如果使用maven-tomcat插件，可以在pom.xml配置url编码。
```xml
  <build>
    <!-- 配置插件 -->
    <plugins>
      <plugin>
        <groupId>org.apache.tomcat.maven</groupId>
        <artifactId>tomcat7-maven-plugin</artifactId>
        <configuration>
          <port>8082</port>
          <path>/</path>
          <uriEncoding>UTF-8</uriEncoding>
        </configuration>
      </plugin>
    </plugins>
  </build>
```

- 后台业务中使用String强行转码。不方便。
```java
key = new String(key.getBytes("iso8859-1"), "utf-8");
```


