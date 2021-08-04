[toc]
## 安装环境 ##
### sdk ###
- [下载地址](https://golang.google.cn/dl/)，之后解压或者安装。
- 设置环境变量：
  - 创建**GOROOT**-sdk根目录：```E:\go\go1.16.5\go```
  - 修改**Path**：添加```%GOROOT%\bin```
  - 创建**GOPATH**-工作空间：```E:\go\gopath```
    - 通常gopath下创建3个目录。bin：打包后的exe文件，pkg：第三方包，src：源文件
- ```go version```测试是否安装成功。

### goland ###
- [下载地址](https://www.jetbrains.com/go/download/other.html)。
- [安装步骤](http://c.biancheng.net/view/6124.html)。

## 开发 ##
### 创建工程 ###
- 创建goModules。<br>![210714.go.new.png](https://img-blog.csdnimg.cn/20210714104450121.png)
- [go mod](http://c.biancheng.net/view/5712.html)相关介绍。类似pom.xml或者build.gradle。

### 引入依赖 ###
- go.mod中管理第三方包版本。
  - [json工具的github](https://github.com/tidwall/gjson)。
```txt
module "demo"

go 1.16

require (
  github.com/tidwall/gjson v1.2.2
)
```

- 根据提示下载包。```go mod download github.com/tidwall/gjson```
  - 注意设置环境变量**GOPROXY**开代理：```GOPROXY=https://goproxy.cn```。[设置步骤](http://c.biancheng.net/view/5712.html)。

### 业务开发 ###
- 工程目录<br>![210714.go.pro.png](https://img-blog.csdnimg.cn/2021071410445096.png)
- ```my_func.go```为工具类
```go
package pack1

import (
	"fmt"
	"github.com/tidwall/gjson"
	"net/http"
)

/**
gjson解析json
*/
func parse(json string, path string) string {
	return gjson.Get(json, path).String()
}

/**
根据url参数path获取预设json中对应路径的值
*/
func HandleHttp(writer http.ResponseWriter, request *http.Request) {
	var json = `{"name":{"first":"Janet","last":"Prichard"},"age":47}`
	var path = request.URL.Query().Get("path")
	if "" != path {
		var pathValue = parse(json, path)
		fmt.Fprint(writer, path+": "+pathValue)
	} else {
		fmt.Fprint(writer, "empty path")
	}
}
```

- ```demo.go```为入口
```go
package main

import (
	"fmt"
	"demo/pack1"
	"net/http"
)

func main() {
	http.HandleFunc("/", pack1.HandleHttp)
	err := http.ListenAndServe("127.0.0.1:8080", nil)
	if err != nil {
		fmt.Println(err)
	}
}
```

### goland调试 ###
- 如果goland调试时缺少包，使用```go mod tidy```引入。
- 如果delve版本低无法debug，[升级delve版本](https://www.jianshu.com/p/63de6073db97)。

### 打包部署 ###
- go.mod同级目录下命令```go install```，打包之后的exe文件在```gopath/bin```下。
- 双击执行，浏览器访问。<br>![210714.go.server.png](https://img-blog.csdnimg.cn/2021071410445098.png)

## 参考资料 ##
- [sdk文档](https://cloud.tencent.com/developer/doc/1101) 
- [go菜鸟教程](https://www.runoob.com/go/go-tutorial.html)
- [Go语言核心36讲](https://time.geekbang.org/column/intro/100013101)
- [网易云课堂视频](https://study.163.com/course/courseLearn.htm?courseId=306002&from=study#/learn/video?lessonId=421012&courseId=306002)
- [gin实战](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI3MjU4Njk3Ng==&action=getalbum&album_id=1362784031968149504&scene=173&from_msgid=2247484393&from_itemidx=1&count=3&nolastread=1#wechat_redirect)