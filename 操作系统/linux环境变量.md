## 环境变量重启后失效 ##
### 现象 ###
- 写在 ```/etc/profile``` 中的环境变量每次重启之后只能用 ```source /etc/profile``` 之后才能加载，非常麻烦。

### 解决 ###
- 在 ```~/.bashrc``` 中添加 ```source /etc/profile``` ，这样每次用户启动bash，则会加载环境变量（[加载顺序](https://blog.csdn.net/qq_40369829/article/details/79917078#初始化顺序)）。<br>![bashrcenv.png](https://img-blog.csdn.net/20180412173200333)
- ~/.bashrc是隐藏文件，需要使用 ```ls -a``` 或者在图形界面使用 ctrl+H 才能列出。

## 环境变量无效 ##
### 现象 ###
- 在环境变量中使用$标识**脚本参数**。如下设置环境变量PWD为空。。。
```
PWD=$12345
```

### 解决 ###
- 加上单引号。
```
PWD='$12345'
```