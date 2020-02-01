[toc]
## 主要流程 ##
- [https://jingyan.baidu.com/article/597035521d5de28fc00740e6.html](https://jingyan.baidu.com/article/597035521d5de28fc00740e6.html)

## 服务启动 ##
- [启动命令](https://www.cnblogs.com/xixihuang/p/5663559.html)。
```txt
mysqld --remove  //删除mysql服务
mysqld --install //安装mysql服务 
mysqld --initialize //一定要初始化 
net start mysql //启动服务
net stop mysql //停止服务
```
- my.ini简单配置。
```txt
[mysql]

# 设置mysql客户端默认字符集

default-character-set=utf8 

[mysqld]

#设置3306端口

port = 3306 

# 设置mysql的安装目录

basedir=E:\mysql\mysql-5.7.17-winx64

# 设置mysql数据库的数据的存放目录

datadir=E:\mysql\mysql-5.7.17-winx64\data

# 允许最大连接数

max_connections=200

# 服务端使用的字符集默认为8比特编码的latin1字符集

character-set-server=utf8

# 创建新表时将使用的默认存储引擎

default-storage-engine=INNODB 

# 设置默认的sql_mode

sql_mode=ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

## mysql配置 ##
- [sql_mode设置](https://blog.csdn.net/Peacock__/article/details/78923479)。[相关问题](https://blog.csdn.net/ch5057997/article/details/78540837)。
- 初始密码为空，不为空参考：[默认密码的查找修改](https://www.cnblogs.com/wolf-sun/p/6543092.html)。
```
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
exit;
```