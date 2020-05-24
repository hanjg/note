[toc]
## Jsonp ##
- Jsonp其实就是一个跨域解决方案。Js跨域请求数据是不可以的，但是**js跨域请求脚本**是可以的。
- 可以把数据封装成一个js语句，做一个方法的等待调用。跨域请求js脚本可以得到此js语句之后，通过回调函数提取数据。

## 实例 ##
### 环境 ###
- 本地端口号8081和8082的两个tomcat，8082的tomcat为门户工程，8081的tomcat作为服务层拦截 /rest/* 请求。
- 两个工程webapp根目录下有相同的json数据,jsonholder.json。
```json
{
  "id": 1,
  "name": "name"
}
```

- 门户工程webapp根目录下有发起ajax调用的页面jsonpTest.jsp。
- 需要同时启动连个tomcat。

### ajax不跨域请求本地数据 ###
- jsonpTest.jsp为：
```jsp
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>JsonpTest</title>
</head>
<body>
</body>
<script src="js/jquery-1.6.4.js" type="text/javascript"></script>
<script>

  function jsonhandle(data) {
    alert("id: " + data.id + "\n" + "name: " + data.name);
  }

  $(document).ready(function () {
    $.ajax({
      type: "get",
      url: "http://localhost:8082/jsonholder.json",
      success: function (data) {
        alert("success");
      }
    });
  })
</script>
</html>
```

- 页面显示弹出，调用成功：<br>![5.jsonp8082.png](http://img-blog.csdn.net/20180323114523579)

### ajax跨域请求数据 ###
- 将ajax的url改为：
```jsp
 url: "http://localhost:8081/jsonholder.json",
```

- 页面无响应，控制台显示访问受限：<br>![5.jsonp8081.png](http://img-blog.csdn.net/20180323114745134)

### jsonp跨域请求脚本 ###
- 服务层中创建控制器：
```java
public class JsonpController {

    @RequestMapping("/jsonp")
    @ResponseBody
    public String showIndex(String callback) {
        return callback + "(" + "{\n"
                + "  \"id\": 1,\n"
                + "  \"name\": \"name\"\n"
                + "}" + ")";

    }

}
```

- 门户工程中的ajax使用jsonp方式：
```js
  $(document).ready(function () {
    $.ajax({
      type: "get",
      url: "http://localhost:8081/rest/jsonp",
      dataType: "jsonp",  //指定服务器返回的数据类型
      jsonp: "callback",   //指定url查询参数名称
      jsonpCallback: "jsonhandle",  //指定回调函数名称
      success: function (data) {
        alert("success");
      }
    });
  })
```

- 页面弹出，调用成功：<br>![5.jsonpalert.png](http://img-blog.csdn.net/20180323115154453)
- 请求URL和响应实体如下：![5.jsonprequest.png](http://img-blog.csdn.net/20180323115244395)<br>![5.jsonpresponse.png](http://img-blog.csdn.net/20180323115329727)