## 开启root用户 ##
- 设置root密码。
```sh
sudo passwd root
```

## 允许root登陆图形界面 ##
- 50-ubuntu.conf添加 ```greeter-show-manual-login=true```
```sh
sudo gedit /usr/share/lightdm/lightdm.conf.d/50-ubuntu.conf
```

## 解决登录错误 ##
- 将/root/.profile中的 ``` mesg n ``` 替换为 ``` tty -s && mesg n``` 。
```sh
sudo gedit /root/.profile
```

