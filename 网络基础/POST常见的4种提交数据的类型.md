[toc]
## post简介 ##
- [post与get的比较](https://blog.csdn.net/qq_40369829/article/details/78476975#http中get和post的区别)。
- Content-Type请求头中规定消息主体的编码方式。
- 消息主体中包含请求数据。

## 常见的Content-Type请求头 ##
### application/x-www-form-urlencoded ###
- 最常见的 POST 提交数据的方式，浏览器的默认提交方式。
- 提交的数据是以 & 分隔的键值对，并进行转码。
```txt
POST http://www.example.com HTTP/1.1<br>
Content-Type: application/x-www-form-urlencoded;charset=utf-8

title=test&sub%5B%5D=1&sub%5B%5D=2&sub%5B%5D=3
```

### multipart/form-data ###
- 一般用来上传文件。
- 使用boundary分隔各个字段的内容（boundary较长，避免与提交数据冲突）。

```txt
POST http://www.example.com HTTP/1.1
Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA

------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="text"

title
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="file"; filename="chrome.png"
Content-Type: image/png

PNG ... content of chrome.png ...
------WebKitFormBoundaryrGKCBY7qhFd3TrwA--
```

### application/json ###
- 请求的消息主体是json字符串。
```txt
POST http://www.example.com HTTP/1.1 
Content-Type: application/json;charset=utf-8

{"title":"test","sub":[1,2,3]}
```

### text/xml ###
- 请求的消息主体是xml片段。

## 参考 ##
- [转载](https://imququ.com/post/four-ways-to-post-data-in-http.html)
- [Web专题](https://imququ.com/post/series.html)
