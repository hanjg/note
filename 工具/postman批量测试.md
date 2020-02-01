[toc]
## postman ##
- postman可以模拟浏览器或者其他应用，发送 **HTTP报文** 。
- ![postmansample.png](https://img-blog.csdn.net/20180417123118834)
- 官网地址：[https://www.getpostman.com/](https://www.getpostman.com/)，包括安装包和文档。

## 环境变量 ##
- postman可以设值环境变量，方便修改发送报文的参数。<br>![postmanenv.png](https://img-blog.csdn.net/20180417135116723)
- ``` {{ }} ``` 可以用来使用环境变量。
- 环境变量可以分为全局变量（global）和普通的环境变量（如上图中的sso）。

## postman脚本 ##
### 请求前 ###
- 在Pre-request Script中可以设置请求之前执行的脚本，通常用来**修改环境变量**。<br>![10.pretest.png](https://img-blog.csdn.net/20180417134941263)

### 响应后 ###
- 在Tests中设置对响应的操作，通常用来**校验**响应是否和预期的一致。<br>![10.tests.png](https://img-blog.csdn.net/20180417135405803)

## 批量测试 ##
- 设置好环境变量，之后在一个Collection中创建报文，设置好脚本之后，使用Runner运行。<br>![10.startrun.png](https://img-blog.csdn.net/20180417135805577)
- 测试结果会以集合的形式显示出来。<br>![10.runresult.png](https://img-blog.csdn.net/20180417135918687)