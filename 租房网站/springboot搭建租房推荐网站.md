[toc]
## 简介 ##
- 由于毕业租房的时候遇到不少坑，想搞一个给刚从学校出来的同学推荐租房信息的网站，目前做出一个雏形。
- github地址：[https://github.com/hanjg/house](https://github.com/hanjg/house)
  - master分支：springboot版
  - ssm分支：SSM版本

### 主要功能 ###
- 目前的功能如下：
  - 持续抓取链家网的租房数据，包括房屋信息和小区信息。
  - 展示所有租房信息。
  - 推送和展示关注的房源的最新信息和爬虫状态。
- 计划增加功能：
  - 从多个网站爬取并汇总信息。
  - 管理关注的小区，可以通过名称，位置等信息设置。
  - 智能推荐房源，综合价格等因素，需要考虑到房屋来源等社会因素。

### 技术选型 ###
- 数据库：msyql
- 后台框架：
  - springboot2
  - mybatis
  - webmagic：爬虫框架抓取网站的数据。
  - websocket推送消息。
- 前台框架：
  - easy-ui：（计划用更加流行的Bootstrap）
  - jsp（继承ssm框架的视图，后计划用效率更高的thymeleaf）

## 主要流程 ##
### webmagic抓取数据 ###
- webmagic中：
  - Downloader负责下载网页。
  - Scheduler负责调度任务的。使用 **url不去重** 的调度器，因为需要重复爬取数据。
  - PageProcessor负责解析下载的网页。
  - Pipeline负责数据的持久化。
  - ProxyPool提供代理。代理无法稳定弃用。<br>![http://webmagic.io/](http://code4craft.github.io/images/posts/webmagic.png)
- 参考[webmagic首页](http://webmagic.io/),[webmagic使用总结](https://blog.csdn.net/yzh_2017/article/details/58092183)。
- webmagic的配置在WebmagicConfig这个配置类中。爬取的服务实现类为CrawlerServiceImpl。
```java
@Configuration
public class WebmagicConfig {

    @Autowired
    private LianjiaConst lianjiaConst;
    @Autowired
    private CrawlerConst crawlerConst;

    @Autowired
    private PageProcessor pageProcessor;
    @Autowired
    private Pipeline pipeline;
    @Autowired
    private HttpClientDownloader downloader;
    @Autowired
    private Scheduler scheduler;
    @Autowired
    private ProxyPool proxyPool;

    @Bean
    public Spider spider() {
        Spider spider = us.codecraft.webmagic.Spider.create(pageProcessor);
        spider.addPipeline(pipeline);
        downloader.setProxyProvider(proxyPool);
        spider.setDownloader(downloader);
        spider.setScheduler(scheduler);
        spider.thread(crawlerConst.getThreadNum());
        return spider;
    }

    @Bean
    public Pipeline pipeline() {
        return new LianjiaDbPipeLine();
    }

    @Bean
    public Scheduler scheduler() {
        return new DuplicateQueueScheduler();
    }

    @Bean
    public PageProcessor pageProcessor() {
        LianjiaPageProcessor pageProcessor = new LianjiaPageProcessor(crawlerConst.getSleepTimes(),
                crawlerConst.getRetryTimes());
        pageProcessor.setCityRentRoot(lianjiaConst.getRentCityRoot());
        pageProcessor.setCity(lianjiaConst.getCityName());
        return pageProcessor;
    }

    @Bean
    public ProxyPool proxyPool() {
        return new ProxyPool();
    }

    @Bean
    public HttpClientDownloader httpClientDownloader() {
        return new HttpClientDownloader();
    }
}
```

### 记录状态的更新 ###
- 房屋和小区均使用状态字段status标志记录的状态，分别为过期、最新、正在更新状态。
```sql
 status TINYINT NOT NULL DEFAULT 2,
```
```java
public enum RecordStatus {
    EXPIRED((byte) 0, "过期"), LATEST((byte) 1, "最新"), UPDATING((byte) 2, "正在更新");

    private Byte status;
    private String state;
}
```

- 主线程管理爬虫，负责爬取数据，新插入的记录或者曾经出现过的记录都会将状态设为**正在更新** 。
- 更新状态线程负责在爬取结束之后将正在更新状态转为最新，曾经最新的状态转为过期。
```xml
  <update id="updateStatus">
    update community
    set status = status - 1
    where status != 0
  </update>
  <update id="updateStatus">
    update renting_house
    set status = status - 1
    where status != 0
  </update>
```

- 两个线程之间在SpiderThreadManager中使用CountDownLatch协调，保证更新线程在爬取结束之后进行。[CountDownLatch使用详解](https://www.cnblogs.com/catkins/p/6021992.html)。
  - 更新线程等待爬取结束。
  - 爬取结束之后唤醒更新线程，主线程等待更新结束。
  - 更新结束之后唤醒主线程。
```java
  public void start(final int repeatTimes, List<String> rootUrls) {
        spiderRunnnig = true;
        int count = repeatTimes;
        while (count-- > 0) {
            updateStatusThreadStart();
            crawlStart(rootUrls);
            try {
                Thread.sleep(10 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        spiderRunnnig = false;
    }

    private void crawlStart(List<String> urlList) {
        try {
            spider.addUrl(urlList.toArray(new String[urlList.size()]));
            spider.start();
            while (true) {
                Thread.sleep(10 * 1000);
                if (spider.getStatus().equals(Status.Stopped)) {
                    break;
                }
            }
            crawlAction.countDown();
            updateStatusAction.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

   private void updateStatusThreadStart() {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    crawlAction.await();

                 ...

                    updateStatusAction.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
```

### 信息的推送 ###
- 使用 **Websocket** 维持网页和浏览器的长连接，当用户打开或者刷新网页时，推送更新的信息至浏览器。
- MyWebSocketHandler重写AbstractWebSocketHandler方法，在连接建立时和接收消息时返回最新的信息。
```java
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        LOGGER.info("websocket connection established......");
        sendLatestNews(session);
    }

    private void sendLatestNews(WebSocketSession session) throws IOException {
        List<RentingHouse> houseList = new ArrayList<>();
        for (String communityName : pusherConst.getPushedCommunities()) {
            houseList.addAll(rentingHouseService.getLatestFavourateHouseList(communityName, lastPushTime));
        }
        String crawlerMessage = getSpiderMessage();
        String houseMessage = getHouseMessage(houseList);
        session.sendMessage(new TextMessage(crawlerMessage + "\n\n" + houseMessage));
        //更新最近推送时间
        lastPushTime = new Date();
        LOGGER.info("last push time: {}", lastPushTime);
    }
    @Override
    public void handleMessage(WebSocketSession session, WebSocketMessage<?> message) throws Exception {
        LOGGER.info("websocket handle text message: {}", message);
        sendLatestNews(session);
    }

```
- websocket参考：[详解教程](https://www.cnblogs.com/jingmoxukong/p/7755643.html)，[websocket整合spring](https://blog.csdn.net/zsg88/article/details/76862495)

## 遇到的问题 ##
### No runnable methods ###
- 单元测试报java.lang.Exception: No runnable methods
- 在src/test/java文件夹下的类中方法添加** @Test注解** 或者将类设置成 **abstract** 。

### net::ERR_CONNECTION_REFUSED ###
- 连接被拒绝，原因有多种。本人遇到磁盘空间耗尽，nginx无法写缓存，从而拒绝连接。
- 解决思路：查看nginx或者tomcat **日志** ，找到对应request的日志。

### 爬取速度慢 ###
- 同一个IP最快可以一秒访问网站两次，否则会被封，解决这一问题最通用的方法是使用代理。
- ProxyServiceImpl中抓取西刺等代理的IP，并且序列化保存在本地，以供爬虫使用。但是抓取的代理极不稳定，验证可用之后使用绝大多数都无法再次访问。由于总记录暂时为1W-2W，平均2h刷新一次，暂时不使用代理。
```java
    private void getProxyFromXici() {
        int currentPage = 1;
        int urlCount = 0;
        while (true) {
            String url = crawlerConst.getXiciRoot() + currentPage;
            LOGGER.info("get proxy from: {}", url);
            try {
                Document document = Jsoup.connect(url).timeout(3 * 1000).get();
                Elements trs = document.getElementsByTag("tr");
                if (trs == null || trs.size() < 1) {
                    break;
                }
                for (int i = 1; i < trs.size(); i++) {
                    try {
                        LOGGER.debug("get url {}", ++urlCount);
                        Elements tds = trs.get(i).getElementsByTag("td");
                        Proxy proxy = new Proxy(tds.get(1).text(), Integer.valueOf(tds.get(2).text()));
                        if (!proxyPool.contain(proxy) && canUse(proxy)) {
                            proxyPool.add(proxy);
                        }
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            LOGGER.error(e.toString());
                        }
                    } catch (Exception e) {
                        LOGGER.error(e.toString());
                    }
                }
            } catch (Exception e) {
                LOGGER.error(e.toString());
            }
            currentPage++;
        }
    }
```

- 参考[验证代理可用](https://blog.csdn.net/zhuyu_deng/article/details/12850685)。

### httpclient超时 ###
- 需要设置两个超时时间间隔。connectTimeout是链接建立的时间，socketTimeout是等待数据的时间或者两个包之间的间隔时间。
```java

    public static boolean isConnServerByHttp(String serverUrl) {// 服务器是否开启
        boolean connFlag = false;
        URL url;
        HttpURLConnection conn = null;
        try {
            url = new URL(serverUrl);
            conn = (HttpURLConnection) url.openConnection();
            conn.setConnectTimeout(3 * 1000);
            if (conn.getResponseCode() == 200) {// 如果连接成功则设置为true
                connFlag = true;
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            conn.disconnect();
        }
        return connFlag;
    }
```

- httpclient请求之后一定要 **close链接** ，否则再次请求会卡住。

- 程序中最好设置connectTimeout、socketTimeout，可以防止阻塞。 
  - 如果不设置connectTimeout会导致，建立tcp链接时，阻塞，假死。
  - 如果不设置socketTimeout会导致，已经建立了tcp链接，在通信时，发送了请求报文，恰好此时，网络断掉，程序就阻塞，假死在那。
- 有时，connectTimeout并不像你想的那样一直到最大时间 
socket建立链接时，如果网络层确定不可达，会直接抛出异常，不会一直到connectTimeout的设定值。[参考](https://blog.csdn.net/wangjun5159/article/details/75138201)。

### TIMESTAMP column with CURRENT_TIMESTAMP ###
- 只能有一个带CURRENT_TIMESTAMP的timestamp列存在。[参考](https://www.jb51.net/article/50878.htm)。

### nginx域名带_字符非法 ###
- ![](https://img-blog.csdnimg.cn/20181117231832882.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2ODk4NDk=,size_16,color_FFFFFF,t_70)

- 配置upstream的不使用 _ 。
```
    upstream local_tomcat {  
        server localhost:8080;
    } 
	改为
    upstream localTomcat {  
        server localhost:8080;
    } 
```

### logback与slf4j的jar冲突 ###
- tomcat启动时异常。该异常的原因是Springboot本身使用logback打印日志，但是项目中其他的组件依赖了slf4j，这就导致了logback与slf4j的jar包之间出现了冲突。
```txt
Exception in thread "main" java.lang.IllegalArgumentException: LoggerFactory is not a Logback LoggerContext but Logback is on the classpath. Either remove Logback or the competing implementation 
```

- 两个jar包二选一：
- 排除slf4j,每个依赖了slf4j的组件都需要加如下标签排除。
```xml
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-log4j</artifactId>
	    <version>1.3.8.RELEASE</version>
	    <exclusions>
	        <exclusion>
	            <groupId>org.slf4j</groupId>
	            <artifactId>slf4j-log4j12</artifactId>
	        </exclusion>
	    </exclusions>
	</dependency>
```

- 排除logback。
```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <exclusions>
        <!--log4j和logback冲突，干掉logback-->
        <exclusion>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
```

- [参考](https://blog.csdn.net/x4609883/article/details/80842155)。
### cookie reject 告警 ###
- 当程序中无需传递cookie值时会出现“Cookie rejected”的警告信息。
```txt
  2018-12-12 18:18:25 [WARN]-[org.apache.http.client.protocol.ResponseProcessCookies] Cookie rejected [select_city="320100", version:0, domain:zufangzi.com, path:/, expiry:Thu Dec 13 18:18:25 CST 2018] Illegal 'domain' attribute "zufangzi.com". Domain of origin: "nj.lianjia.com"
```

- 如使用httpclient，忽略cookie即可,[参考](https://blog.csdn.net/wolfjin/article/details/54767161)。
```java
RequestConfig globalConfig = RequestConfig.custom().setCookieSpec(CookieSpecs.IGNORE_COOKIES).build();  
CloseableHttpClient client = HttpClients.custom().setDefaultRequestConfig(globalConfig).build();  
HttpGet request = new HttpGet(url);  
CloseableHttpResponse response = client.execute(request);  
```

- 如使用webmagic，分析源码，需要设置site的disablecookiemanagement这个属性。<br>![181212.sitecookie.png](https://img-blog.csdnimg.cn/20181212203519980.png)
```java
    public LianjiaPageProcessor(int sleepTime, int retryTimes) {
        this.site = Site.me().setRetryTimes(retryTimes).setSleepTime(sleepTime).setDisableCookieManagement(true);
    }
```

## springboot和ssm区别 ##
- 默认不支持jsp，需要添加的话，参考：[springboot项目添加jsp支持](https://www.cnblogs.com/yuxiaona/p/7677717.html)
- [mybatis的整合](https://blog.csdn.net/Winter_chen001/article/details/80010967)
  - 无需手动配置sqlSessionFactory，自动配置的factory可以应对大多数情况，否则某些自动配置的factory加载不了yml配置，如:
```yml
mybatis:
  mapper-locations: classpath:mapper/*.xml 
```

- [从SpringMVC迁移到Springboot](http://ju.outofmemory.cn/entry/339386)

## springboot2和之前版本的区别 ##
- 版本要求：
  - java8以上
  - ​Tomcat升级至8.5
  - Flyway升级至5
  - Hibernate升级至5.2
  - Thymeleaf升级至3
- 配置属性<br>![181210.sb2.png](https://img-blog.csdnimg.cn/20181210105401623.png)
 