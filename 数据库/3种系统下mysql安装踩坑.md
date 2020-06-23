[toc]
## 本地window ##
### 主要流程 ###
- [https://jingyan.baidu.com/article/597035521d5de28fc00740e6.html](https://jingyan.baidu.com/article/597035521d5de28fc00740e6.html)

### 服务启动 ###
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

### mysql配置 ###
- [sql_mode设置](https://blog.csdn.net/Peacock__/article/details/78923479)。[相关问题](https://blog.csdn.net/ch5057997/article/details/78540837)。
- 初始密码为空，不为空参考：[默认密码的查找修改](https://www.cnblogs.com/wolf-sun/p/6543092.html)。
```
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
exit;
```

## 远程ubuntu ##
### 安装mysql服务器 ###
- 使用管理员用户登录。
- 安装mysql客户端。
```sh
sudo apt-get install mysql-server
```

- 检查mysql状态，socket处于listen状态。
```sh
sudo netstat -tap | grep mysql
```
![](https://img-blog.csdn.net/20180628212019641)

### 修改mysql监听端口 ###
- 修改mysql监听ip为所有ip。在 ```/etc/mysql/my.cnf``` 中将bind-address改为 ```0.0.0.0``` 。
- 重启mysql。
```sh
service mysql restart
```
- 查询监听ip。
```sh
netstat -ano | grep 3306
```
![180628.mysqllistenip.png](https://img-blog.csdn.net/20180628213012125)

### 创建远程登录账号 ###
- 登录mysql。
```sh
mysql -u root -p
```
- 创建远程登录账号。
```sh
grant all privileges on *.* to '用户'@'ip' identified by '密码' with grant option;
flush privileges;
```

### 配置安全组规则 ###
- 配置安全组规则：云服务器控制台->网络和安全组->安全组配置，开放mysql端口。<br>![180628.group.png](https://img-blog.csdn.net/20180628213605839)

## 远程centos ##
- [参考](https://juejin.im/post/5c088b066fb9a049d4419985)。

### 安装mysql ###
- 添加仓库。
```sh
sudo rpm -ivh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```

- 确认添加成功。
```sh
sudo yum repolist all | grep mysql | grep enabled
```
```sh
mysql-connectors-community/x86_64  MySQL Connectors Community    enabled:     51
mysql-tools-community/x86_64       MySQL Tools Community         enabled:     63
mysql57-community/x86_64           MySQL 5.7 Community Server    enabled:    267
```

- 安装。
```sh
sudo yum -y install mysql-community-server
```

### 启动mysql ###
```sh
sudo systemctl start mysqld
```

### 配置mysql ###
- 修改端口号。路径：```/etc/my.cnf```
```	
[mysqld]
port = 3307
```

- 配置密码。默认密码位置，通过命令显示。``` /var/log/mysqld.log ```
```sh
cat /var/log/mysqld.log | grep -i 'temporary password'
```

- 如果用环境变量配置密码，需要执行命令立刻生效。
```
source /etc/profile
```