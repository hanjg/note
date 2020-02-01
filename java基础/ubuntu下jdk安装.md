## 环境 ##
- ubuntu14下安装java8。

## 步骤 ##
- 下载安装包，注意操作系统类型和位数，地址：[http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- 切换为root。
```cmd
sudo su
```

- 创建java安装目录。
```cmd
mkdir /usr/java
```

- 将下载的压缩包解压到java目录。
```cmd
tar -zxvf jdk-8u161-linux-x64.tar.gz -C /usr/java
```

- 在/etc/profile中添加环境变量。
```cmd 
gedit /etc/profile
```
```txt
export JAVA_HOME=/usr/java/jdk1.8.0_161
export CLASSPATH=$JAVA_HOME/lib/

export PATH=$PATH:$JAVA_HOME/bin
```

- 重启或者重新加载profile。
```cmd
shutdown -r now
source /etc/profile
```

- 检查版本。
```cmd
java -version
```
![jdkversion.png](http://img-blog.csdn.net/20180411172140681)

```