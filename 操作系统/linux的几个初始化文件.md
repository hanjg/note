[toc]
## 初始化文件 ##
### 全局配置 ###
#### /etc/profile ####
- 在登录时,操作系统定制用户环境时使用的第一个文件，此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行。并从/etc/profile.d目录的配置文件中搜集shell的设置。这个文件一般就是调用/etc/bash.bashrc文件。


#### /etc/bashrc ####
- 或者/etc/bash.bashrc（ubuntu下）.
- 为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取。


#### /etc/environment ####
- 在登录时操作系统使用的第二个文件,系统在读取你自己的profile前,设置环境文件的环境变量。


### 个人配置 ###
#### ~/.profile ####
- 在登录时用到的第三个文件 是.profile文件,每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次!默认情况下,他设置一些环境变量,执行用户的.bashrc文件。

#### ~/.bashrc ####
- 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该该文件被读取。不推荐放到这儿，因为每开一个shell，这个文件会读取一次，效率上讲不好。

#### ~/.bash_profile ####
- 每个用户都可使用该文件输入专用于自己 使用的shell信息,当用户登录时,该文件仅仅执行一次!默认情况下,他设置一些环境变量,执行用户的.bashrc文件。~/.bash_profile 是交互式、login 方式进入 bash 运行的~/.bashrc是交互式 non-login 方式进入 bash 运行的通常二者设置大致相同，所以通常前者会调用后者。

#### ~./bash_login ####
- 不推荐使用这个，这些不会影响图形界面。而且.bash_profile优先级比bash_login高。当它们存在时，登录shell启动时会读取它们。

#### ~/.bash_logout ####
- 当每次退出系统(退出bash shell)时,执行该文件.


#### ~/.pam_environment ####
- 用户级的环境变量设置文件。

## 初始化顺序 ##
- 图形模式登录时，顺序读取：/etc/profile和~/.profile 
- 图形模式登录后，打开终端时，顺序读取：/etc/bash.bashrc和~/.bashrc 
- 文本模式登录时，顺序读取：/etc/bash.bashrc，/etc/profile和~/.bash_profile

## 注意 ##
- 环境变量尽量在profile中设置，如果在bashrc中设置或者是进行一些操作，万一出错很可能会导致bash不能使用。
- 特别是操作 **/etc/bash.bashrc** 文件一定小心，如果错误无法使用bash，就算是用图形界面登录也由于没有root权限，无法还原文件，只能读取。

## 参考 ##
- [http://blog.51cto.com/19055/1144600](http://blog.51cto.com/19055/1144600)
- [https://blog.csdn.net/Field_Yang/article/details/51087178](https://blog.csdn.net/Field_Yang/article/details/51087178)
- [https://www.linuxidc.com/Linux/2016-09/135476.htm](https://www.linuxidc.com/Linux/2016-09/135476.htm) 