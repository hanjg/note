[toc]
## 安装 ##
### es安装 ###
- 下载：[官网](https://www.elastic.co/cn/downloads/elasticsearch)、[历史版本](https://www.elastic.co/cn/downloads/past-releases)，或者[国内镜像](https://www.newbe.pro/Mirrors/Mirrors-Elasticsearch/)。
- [教程](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/getting-started.html)。
- 配置 ```config/elasticsearch.yml```。[参考](https://www.jianshu.com/p/88f0546d5955)。
```yml
cluster.name: my-cluster
node.name: my-node1
network.host: 0.0.0.0
http.port: 9200
cluster.initial_master_nodes: ["my-node1"]
```
- ```bin/elasticsearch.bat``` 启动。
- 访问： ```http://localhost:9200```。

### kibana安装 ###
- 下载：[官网](https://www.elastic.co/downloads/kibana)，或者[国内镜像](https://www.newbe.pro/Mirrors/Mirrors-Kibana/)。
- [教程](https://www.cnblogs.com/chenqionghe/p/12503181.html)。
- 配置 ```config/kibana.yml```。
```yml
server.port: 5601
server.host: "localhost"
server.name: "my-kibana-server"
elasticsearch.hosts: ["http://localhost:9200"]
i18n.locale: "zh-CN"
```
- ```bin/kibana.bat``` 启动。
- 访问： ```http://localhost:5601``` 。

## 命令行使用 ##
### 索引操作 ###
- 查询所有索引。
```
GET /_cat/indices
```

- 创建索引。索引字段不能删除和修改类型，只能新增。
```
PUT /second_hand_house
{
    "mappings" : {
      "properties" : {
        "houseCode" : {
          "type" : "text"
        },
        "title" : {
          "type" : "text"
        },
        "priceTotal" : {
          "type" : "integer"
        },
        "unitPrice" : {
          "type" : "integer"
        },
        "area" : {
          "type" : "double"
        },
        "fromTime" : {
          "type" : "long"
        },
        "toTime" : {
          "type" : "long"
        }
      }
    },
    "settings" : {
      "index" : {
        "refresh_interval" : "1s",
        "number_of_shards" : "3",
        "number_of_replicas" : "1"
      }
    }
}
```

- 查询索引结构。
```
GET second_hand_house
```

- 删除索引。
```
DELETE second_hand_house
```

### 文档操作 ###
- 创建文档。
```
PUT /second_hand_house/_doc/001?routing=SH001
{
	"houseCode" : "SH001",
	"title" : "房屋1",
	"priceTotal" : 20000000,
	"unitPrice" : 100000,
	"area" : 200.00,
	"fromTime" : 1597680310,
	"toTime" : 1597690308
}

PUT /second_hand_house/_doc/002?routing=SH002
{
	"houseCode" : "SH002",
	"title" : "房屋2",
	"priceTotal" : 10000000,
	"unitPrice" : 100000,
	"area" : 100.00,
	"fromTime" : 1596680310,
	"toTime" : 1596690308
}
```

- 更新文档。
```
PUT /second_hand_house/_doc/001?routing=SH001
{
	"houseCode" : "SH001",
	"title" : "房屋123",
	"priceTotal" : 20000000,
	"unitPrice" : 100000,
	"area" : 200.00,
	"fromTime" : 1597680310,
	"toTime" : 1597690308
}
```

- 删除文档。
```
DELETE second_hand_house/_doc/001
```

- 查询文档。
```
GET second_hand_house/_doc/001?routing=SH001
```

### 搜索操作 ###
- 搜索全部文档。
```
GET second_hand_house/_search
{
  "query": {
    "match_all": {}
  }
}
```

- 条件查找。
```
GET second_hand_house/_search?size=10&from=0
{
  "query": {
    "match":{
      "title": "房"
    }
  }
}
```

- [深度分页查找](https://blog.csdn.net/u011228889/article/details/79760167)。

## java客户端使用 ##
- [参考](https://my.oschina.net/GinkGo/blog/1853345#h3_11)。

### 初始化 ###
- 引入依赖。es客户端版本交给springboot管理，否则容易冲突。
```xml
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>elasticsearch-rest-high-level-client</artifactId>
      <!--<version>${elastic.client.version}</version>-->
    </dependency>
```

- 初始化Bean。
```java
    @Bean
    public RestHighLevelClient restHighLevelClient() {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http")));
        return client;
    }
```

### 文档操作 ###
- 创建、更新文档。
```java
    Map<String, Object> map = BeanUtils.objectToMap(secondHandHouseEO);
    IndexRequest request = new IndexRequest("second_hand_house", "_doc", secondHandHouseEO.getHouseCode())
            .source(map);
    request.routing(secondHandHouseEO.getHouseCode());
	IndexResponse indexResponse = client.index(request, RequestOptions.DEFAULT);
    if (indexResponse.getResult() == Result.CREATED || indexResponse.getResult() == Result.UPDATED) {
        return true;
    }
    ReplicationResponse.ShardInfo shardInfo = indexResponse.getShardInfo();
    if (shardInfo.getFailed() > 0) {
        for (ReplicationResponse.ShardInfo.Failure failure : shardInfo.getFailures()) {
            String reason = failure.reason();
            LOGGER.error("put es error, {}", reason);
        }
        return false;
    }
	return true;        
```

- 删除文档。
```java
    DeleteRequest request = new DeleteRequest("second_hand_house", "_doc", houseCode);
    request.routing(houseCode);
    DeleteResponse deleteResponse = client.delete(request, RequestOptions.DEFAULT);
    ReplicationResponse.ShardInfo shardInfo = deleteResponse.getShardInfo();
    if (shardInfo.getFailed() > 0) {
        for (ReplicationResponse.ShardInfo.Failure failure : shardInfo.getFailures()) {
            String reason = failure.reason();
            LOGGER.error("delete es error, {}", reason);
        }
        return false;
    }
    return true;
```

### 搜索操作 ###
- 分页查找。
```java
	SearchRequest request = new SearchRequest("second_hand_house");
	SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
	sourceBuilder.from(offset);
	sourceBuilder.size(limit);
	sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
	request.source(sourceBuilder);
	
	SearchResponse searchResponse = client.search(request, RequestOptions.DEFAULT);
	List<SecondHandHouseEO> list = new ArrayList<>();
	for (SearchHit hit : searchResponse.getHits()) {
	    Map<String, Object> map = hit.getSourceAsMap();
	    list.add(BeanUtils.mapToObject(map, SecondHandHouseEO.class));
	}
```

- 匹配查找。
```java
	SearchRequest request = new SearchRequest("second_hand_house");
	SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
	sourceBuilder.from(offset);
	sourceBuilder.size(limit);
	sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
    MatchQueryBuilder queryBuilder = new MatchQueryBuilder("title", title);
    sourceBuilder.query(queryBuilder);
	request.source(sourceBuilder);
	
	SearchResponse searchResponse = client.search(request, RequestOptions.DEFAULT);
	List<SecondHandHouseEO> list = new ArrayList<>();
	for (SearchHit hit : searchResponse.getHits()) {
	    Map<String, Object> map = hit.getSourceAsMap();
	    list.add(BeanUtils.mapToObject(map, SecondHandHouseEO.class));
	}
```

