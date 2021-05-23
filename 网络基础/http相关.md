[toc]
## HTTP方法的幂等性 ##
- HTTP方法的幂等性是指一次和多次请求某一个资源应该具有同样的副作用。

## http中GET和POST的区别 ##
### 原理上 ###
 - GET用于信息获取。
 - POST表示可能修改变服务器上的资源的请求。
### 请求数据位置 ###
 - GET请求的数据会附在URL之后的查询字符串中，以?分割URL和传输数据，参数之间以&相连。
 - POST把提交的数据则放置在是HTTP报文的body中，编码方式不限，常用application/x-www-form-urlencoded，application/json，multipart/form-data，text/xml等。
### 请求数据大小 ###
 - 因为GET是通过URL提交数据，而HTTP协议规范没有对URL长度进行限制，但url长度受特定的浏览器及服务器的限制（1024字节）。
 - 理论上讲，POST是没有大小限制的，HTTP协议规范也没有进行大小限制，起限制作用的是服务器处理程序的能力（80K/100K）。
### 安全性 ###
 - 使用GET方法时，浏览器可能会缓存地址等信息，留下历史记录。
 - POST方法则不会进行缓存。
### 效率 ###
- get发送一次报文，post发送两次，get效率高。
- [http://www.oschina.net/news/77354/http-get-post-different](http://www.oschina.net/news/77354/http-get-post-different "http://www.oschina.net/news/77354/http-get-post-different")

## OSI七层模型以及TCP/IP四层模型 ##
 - TCP头：TCP数据报，包含源端和目的端的端口号，用于寻找发端和收端的应用进程；
 - IP头：用于寻找网络中目的主机在逻辑网络中的位置；
 - LLC头：负责识别网络层协议，然后对它们进行封装。LLC报头告诉数据链路层一旦帧被接收到时，应当对数据包做何处理。它的工作原理是这样的：主机接收到帧并查看其LLC报头，以找到数据包的目的地，比如说，在网络层的IP协议。
 - MAC头：用于寻找主机在网络设备中的位置；
 - TCP（传输控制协议，传输效率低，可靠性强，用于传输可靠性要求高，数据量大的数据），UDP（用户数据报协议，与TCP特性恰恰相反，用于传输可靠性要求不高，数据量小的数据，如QQ聊天数据就是通过这种方式传输的）。 
 - [http://www.cnblogs.com/commanderzhu/p/4821555.html](http://www.cnblogs.com/commanderzhu/p/4821555.html)

## TCP三次握手四次挥手 ##
### 三次握手 ###
 - SYN 1,SEND SEQ x,client SYN_SEND
 - SYN 1,ACK 1,SEND SEQ y,REC SEQ x+1,server SYN_RECV
 - SYN 0,ACK 1,REC SEQ y+1,client+server ESTABLISHED
![这里写图片描述](http://img.blog.csdn.net/20171108114528929?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfNDAzNjk4Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 四次挥手 ###
 - SEND SEQ x+2,REC SEQ y+1,client FIN_WAIT_1
 - REC SEQ =SEND SEQ +1,client FIN_WAIT_2
 - SEND SEQ y+1,server LAST_ACK
 - REC SEQ = SEND SEQ+1,client TIME_WAIT server CLOSED client等待2MSL后依然没有回复，则证明服务端已正常关闭，客户端此时关闭连接进入CLOSED状态。
![这里写图片描述](http://img.blog.csdn.net/20171108114553927?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfNDAzNjk4Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## TCP三次握手四次挥手原因 ##
### 三次握手 ###
- 防止A已失效的连接请求传到B，产生错误，所以A需要对B的确认进行确认。
### 四次挥手 ###
- TCP全双工通信，前两次挥手关闭A到B的通信，这个时候B到A仍然可以发送数据。

## TCP为什么TIME_WAIT状态还需要等2MSL(Maximum Segment Lifetime 报文最长存活时间)后才能返回到CLOSED状态 ##
#### 保证A发送的最有一个ACK报文段能够到达B ####
- 这个ACK报文段有可能丢失，因而使处在LAST-ACK状态的B收不到对已发送的FIN和ACK 报文段的确认，B会超时重传这个FIN和ACK报文段，而A就能在2MSL时间内收到这个重传的ACK+FIN报文段，接着A重传一次确认。
#### 防止已失效的连接请求报文段出现在新的连接中 ####
- A在发送完最有一个ACK报文段后，再经过2MSL，就可以使本连接持续的时间内所产生的所有报文段都从网络中消失。

## http和https的区别 ##
- ![这里写图片描述](http://img.blog.csdn.net/20171108114608863?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfNDAzNjk4Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1. HTTP 的URL以http://开头，而 HTTPS 的URL以https://开头。
3. HTTP 标准端口是 80 ，而 HTTPS 的标准端口是 443。
4. 在 OSI 网络模型中，HTTP 工作于应用层，而 HTTPS 工作在传输层。
2. HTTP 是不安全的，而 HTTPS 是安全的。
5. HTTP 无需加密，而 HTTPS 对传输的数据进行加密。
6. HTTP 无需证书，而 HTTPS 需要认证证书。

## https实现原理 ##
- C/S连接过程：
  - C/S http握手。
  - S发送**CA证书**给C。
    - 证书为CA机构颁发。
    - 包含**S的公钥**、组织信息、签发的CA信息、有效时间、证书序列号、**签名**(CA的明文hash之后用**CA的私钥**加密)。
  - C收到证书后用相同的函数hash，和**CA公钥**解密签名对比验证CA的有效性。
  - C用**S的公钥**加密之后业务通信的**对称加密算法和对称秘钥**。
  - S用**S的私钥**解密获得对称秘钥。
  - C/S 使用对称秘钥加密通信。
- [参考1](http://blog.csdn.net/whatday/article/details/38147103)，[参考2](https://www.cnblogs.com/yunlongaimeng/p/9417276.html)

## session和cookie的区别 ##
### 存放地点 ###
 - cookie数据存放在客户的浏览器上，session数据放在服务器上。
### 安全性 ###
 - cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗，考虑到安全应当使用session。
### 服务器性能 ###
 - session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能，考虑到减轻服务器性能方面，应当使用COOKIE。
### 使用方式 ###
 - 将登陆信息等重要信息存放为SESSION，其他信息如果需要保留，可以放在COOKIE中
### 联系 ###
- cookie中保存session id。
- 如果cookie被人为的禁止，有两种解决方法：<br>1、URL重写，就是把session id直接附加在URL路径的后面；<br>2、服务器会自动修改表单，添加一个隐藏字段，以便在表单提交时能够把session id传递回服务器。
### 参考 ###
- [http://www.cnblogs.com/shiyangxt/archive/2008/10/07/1305506.html](http://www.cnblogs.com/shiyangxt/archive/2008/10/07/1305506.html)

## dos攻击 ##
[http://blog.csdn.net/codeforme/article/details/8749832](http://blog.csdn.net/codeforme/article/details/8749832 "http://blog.csdn.net/codeforme/article/details/8749832")

## HTTP状态码 ##
- 2** 成功：操作被成功接受或者处理。
- 3** 重定向：需要进一步操作已完成请求。
- 4** 客户端错误：包含语法错误或者无法完成的请求。
- 5** 服务器错误：服务器在处理请求的过程中发生错误。
- [https://www.w3cschool.cn/http_api_guide/ny61rozt.html](https://www.w3cschool.cn/http_api_guide/ny61rozt.html "https://www.w3cschool.cn/http_api_guide/ny61rozt.html")

## 缓存相关的http首部 ##
### 通用缓存首部 ###
1. Cache-Control：用于岁报文传送缓存指示。
2. Pragma：类似前者，不专用于缓存。
### 实体缓存首部 ###
1. ETag：与此实体相关的实体标记。
2. Expires：指定过期时间，实体不再有效，要从原始的源端再次获取次实体的日期和时间。
3. Last-Modified：这个实体最后一个被修改的日期和时间。
### 条件首部 ###
#### 满足条件后返回对象 ####
1. If-Modified-Since:<data>，如果从指定日期之后文档被修改过，执行请求的方法。
2. If-None-Match:<tags>，如果已缓存的标签与服务器文档中的标签有所不同，执行请求的方法。

