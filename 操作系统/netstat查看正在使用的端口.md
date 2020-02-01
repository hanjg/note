## 参数 ##
```cmd
　-t : 指明显示TCP端口
　-u : 指明显示UDP端口
　-l : 仅显示监听套接字(所谓套接字就是使应用程序能够读写与收发通讯协议(protocol)与资料的程序)
　-p : 显示进程标识符和程序名称，每一个套接字/端口都属于一个程序。
　-n : 不进行DNS轮询，显示IP(可以加速操作)
```

## 实例 ##
- 与grep结合可查看某个具体端口及服务情况。
```cmd
netstat -ntlp   //查看当前所有tcp端口·
netstat -ntulp |grep 80   //查看所有80端口使用情况·
netstat -an | grep 3306   //查看所有3306端口使用情况·
```

[http://blog.csdn.net/q361239731/article/details/53180126](http://blog.csdn.net/q361239731/article/details/53180126)