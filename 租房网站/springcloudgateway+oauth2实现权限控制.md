[toc]
## 鉴权服务 ##
- [OAuth 2.0 的四种方式](http://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html)
  - 授权码（避免令牌泄露）：
    - A请求授权码，在B登录后返回A授权码。
    - A在后端用授权码向B请求令牌。
  - 隐藏式。前端直接返回令牌。
  - 密码式。A用在B的用户名密码后端请求令牌。
  - 凭证式。A获得公用令牌。
- [OAuth2实现分析](https://blog.csdn.net/bluuusea/aricle/details/80284458)

### 基础配置 ###
- 新建house-oauth模块，依赖oauth2。
```xml
  <dependencies>
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-common</artifactId>
      <version>${house.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-oauth2</artifactId>
    </dependency>
  </dependencies>
```

- bootstrap.properties初始化依赖的nacos配置。
```properties
spring.application.name=house-oauth
spring.profiles.active=oauth-application
spring.cloud.nacos.config.namespace=${HOUSE_ENV}
spring.cloud.nacos.config.group=house
spring.cloud.nacos.config.server-addr=${NACOS_ADDR}
spring.cloud.nacos.config.prefix=house
spring.cloud.nacos.config.file-extension=yaml
```

- nacos的house-oauth-application.yaml中配置端口为6001。
```yaml
server:
  port: 6001

spring:
  application:
    name: oauth2-server
```

### 鉴权配置 ###
- 初始化基础配置：认证管理器、加密方式。
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        //默认的认证操作
        return super.authenticationManagerBean();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        //加密器
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //允许匿名访问所有接口 主要是 oauth 接口
        http.authorizeRequests().antMatchers("/**").permitAll();
    }
}
```

- 继承spring-security的UserDetailsService，管理用户密码、角色信息。
```java
@Service
public class UserServiceImpl implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //TODO 用户持久化
        if ("admin".equals(username)) {
            String role = "ROLE_ADMIN";
            List<SimpleGrantedAuthority> authorities = new ArrayList<>();
            authorities.add(new SimpleGrantedAuthority(role));
            String password = passwordEncoder.encode("123456");
            return new User(username, password, authorities);
        }
        throw new UsernameNotFoundException("no user");
    }
}
```

- 配置默认资源（即接口）管理服务。如果不使用默认实现，[可参考](https://juejin.im/post/6844904095942180878#heading-3)。
```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
}
``` 

- 配置授权服务。
```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    public PasswordEncoder passwordEncoder;
    @Autowired
    public UserDetailsService userDetailsService;
    @Autowired
    private AuthenticationManager authenticationManager;

    @Override
    public void configure(final AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        //TODO token持久化
        //配置授权服务策略
        endpoints.authenticationManager(authenticationManager).userDetailsService(userDetailsService);
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        //TODO 客户端持久化
        //配置网关服务的用户名密码，仅网关服务可作为客户端可访问oauth服务
        clients.inMemory()
                .withClient("gateway-client").secret(passwordEncoder.encode("123456"))
                .authorizedGrantTypes("refresh_token", "authorization_code", "password")
                .accessTokenValiditySeconds(24 * 3600)
                .scopes("all");
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        //允许客户端发送表单来进行权限认证来获取令牌
        //只允许认证的客户端，比如网关服务才可以获取和校验token
        security.allowFormAuthenticationForClients()
                .checkTokenAccess("isAuthenticated()")
                .tokenKeyAccess("isAuthenticated()");
    }
}
```

## 网关服务 ##
- [全局过滤器拦截并处理请求](https://cloud.tencent.com/developer/article/1650118)。
- [流行网关对比](http://blog.itpub.net/31562044/viewspace-2651041/)。<br>![](http://img.blog.itpub.net/blog/2019/07/18/c8c9996b62d61c27.jpeg?x-oss-process=style/bb)
- [网关的理解](https://www.cnblogs.com/savorboard/p/api-gateway.html)：
  - 系统网关+业务网关。

### 基础配置 ###
- 新建house-gateway服务，引入spring cloud gateway 依赖。
  - spring-boot-starter-web和gateway冲突，需要去除。
```xml
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
      <groupId>com.babyjuan</groupId>
      <artifactId>house-common</artifactId>
      <version>${house.version}</version>
      <exclusions>
        <exclusion>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-web</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
  </dependencies>
```

- bootstrap.properties初始化依赖的nacos配置。
```properties
spring.application.name=house-gateway
spring.profiles.active=gateway-application
spring.cloud.nacos.config.namespace=${HOUSE_ENV}
spring.cloud.nacos.config.group=house
spring.cloud.nacos.config.server-addr=${NACOS_ADDR}
spring.cloud.nacos.config.prefix=house
spring.cloud.nacos.config.file-extension=yaml
```

- nacos的house-gateway-application.yaml中
  - 配置端口为9080。
  - 获取token的请求(/oauth/token开头)转发到鉴权服务器。
  - 业务请求转发到业务网关。
```yaml
server:
  port: 9080

service:
  url:
    house: http://localhost:8080
    oauth: http://localhost:6001

spring:
  cloud:
    gateway:
      routes:
        # 路由的ID
        - id: oauth
          # 匹配后路由地址
          uri: ${service.url.oauth}
          predicates:
            - Path=/oauth/token
        # 路由的ID
        - id: house
          # 匹配后路由地址
          uri: ${service.url.house}
          predicates:
            - Path=/house/**
```

### 网关过滤器配置 ###
- 请求头中赋予网关服务合法客户端身份。对应鉴权服务AuthorizationServerConfig中gateway-client的用户名密码。
```java
@Configuration
public class GatewayClientFilter implements GlobalFilter, Ordered {

    private static final String GATEWAY_CLIENT_AUTHORIZATION = "Basic " +
            Base64.getEncoder().encodeToString("gateway-client:123456".getBytes());

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        //网关身份
        ServerHttpRequest.Builder builder = exchange.getRequest().mutate();
        builder.header("Authorization", GATEWAY_CLIENT_AUTHORIZATION);
        return chain.filter(exchange.mutate().request(builder.build()).build());
    }

    @Override
    public int getOrder() {
        return 1;
    }
}
``` 

### 权限校验过滤器配置 ###
- 对需要鉴权的url，调用鉴权服务校验身份，通过则网关转发，否则直接返回。
```java
@Configuration
public class AccessGatewayFilter implements GlobalFilter, Ordered {

    @Autowired
    private AuthClient authClient;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String url = request.getPath().value();
        if (authClient.hasPermissionControl(url)) {
            if (authClient.accessable(request)) {
                return chain.filter(exchange);
            }
            return unauthorized(exchange);
        }
        return chain.filter(exchange);
    }

    private Mono<Void> unauthorized(ServerWebExchange serverWebExchange) {
        serverWebExchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        DataBuffer buffer = serverWebExchange.getResponse()
                .bufferFactory().wrap(HttpStatus.UNAUTHORIZED.getReasonPhrase().getBytes());
        return serverWebExchange.getResponse().writeWith(Flux.just(buffer));
    }

    @Override
    public int getOrder() {
        return 2;
    }
}
```

- token从url的请求参数获取。
- 校验token的鉴权服务url后缀为```/oauth/check_token```。
```java
@Component
public class AuthClient {

    private static final Logger LOGGER = LoggerFactory.getLogger(AuthClient.class);

    private RestTemplate restTemplate = new RestTemplate();

    @Value("#{'${service.url.oauth}'+'/oauth/check_token'}")
    private String checkTokenUrl;

    public boolean hasPermissionControl(String url) {
        return url.startsWith("/house");
    }

    public boolean accessable(ServerHttpRequest request) {
        String token = request.getQueryParams().getFirst("token");
        UriComponentsBuilder builder = UriComponentsBuilder.fromHttpUrl(checkTokenUrl).queryParam("token", token);
        URI url = builder.build().encode().toUri();

        HttpEntity<?> entity = new HttpEntity<>(request.getHeaders());

        try {
            ResponseEntity<TokenInfo> response = restTemplate.exchange(url, HttpMethod.GET, entity, TokenInfo.class);
            LOGGER.info("oauth request: {}, response body: {}, reponse status: {}",
                    entity, response.getBody(), response.getStatusCode());
            return response.getBody() != null && response.getBody().isActive();
        } catch (RestClientException e) {
            LOGGER.error("oauth failed.", e);
        }
        return false;
    }
}
```

## 接口调试 ##
- 从网关获取token。<br>![201010.gettoken.png](https://img-blog.csdnimg.cn/20201011002113492.png)
- 带token从网关访问业务通过鉴权。<br>![201010.checktoken.png](https://img-blog.csdnimg.cn/20201011002113505.png)

## 前端适配 ##
- 登录时从后端获取token，暂存在store和cookies中。
```js
const actions = {
  // user login
  login({ commit }, userInfo) {
    const { username, password } = userInfo
    return new Promise((resolve, reject) => {
      login({ username: username.trim(), password: password, grant_type: "password" }).then(response => {
        const data = response
        commit('SET_TOKEN', data.access_token)
        commit('SET_NAME', username.trim())
        setToken(data.access_token)
        resolve()
      }).catch(error => {
        reject(error)
      })
    })
  },
}
const mutations = {
  SET_TOKEN: (state, token) => {
    state.token = token
  },
  SET_NAME: (state, name) => {
    state.name = name
  }
}
export function setToken(token) {
  return Cookies.set(TokenKey, token)
}
```

- axios拦截器获取store中的token，拼接在查询参数里。
```js
service.interceptors.request.use(
  config => {
    // do something before request is sent

    if (store.getters.token) {
      let params = config.params;
      if (params == null) {
        params = {};
      }
      params.token = getToken();
      config.params = params;
    }
    return config
  },
  error => {
    // do something with request error
    console.log(error) // for debug
    return Promise.reject(error)
  }
)
export function getToken() {
  return Cookies.get(TokenKey)
}
```