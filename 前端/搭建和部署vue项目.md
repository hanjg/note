[toc]
## 环境 ##
### nodejs ###
- [安装nodejs](https://www.runoob.com/nodejs/nodejs-install-setup.html)。
- [卸载nodejs](https://blog.csdn.net/ZyhMemory/article/details/93159084)。用于安装失败，清除残留。

### vue ###
- vue[运行流程](https://juejin.im/post/5ba9d5cce51d450e805b59b0)。<br>![](https://user-gold-cdn.xitu.io/2018/9/25/1660fa04ff04f167?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 开发项目 ##
### vue-cli ###
- [cnpm安装vue-cli](https://blog.csdn.net/qq_37568942/article/details/80808148)。npm安装成功率较低。
```txt
npm install -g cnpm --registry=https://registry.npm.taobao.org
cnpm install vue
cnpm install -g vue-cli
```

- 初始化项目。
```txt
vue init webpack vue
```

- 启动项目。
```txt
npm run dev
```

- [卸载vue-cli](https://www.html.cn/qa/vue-js/14780.html)。防止安装失败。
```txt
npm uninstall vue-cli -g 
```	

### 模板 ###
- [https://github.com/PanJiaChen/vue-admin-template](https://github.com/PanJiaChen/vue-admin-template)


## 部署项目 ##
- [安装nginx](https://www.jianshu.com/p/96691511295f)。
- [前端部署在nginx](https://www.jianshu.com/p/ac2d7d502999)。生成的dist目录，复制到nginx目录下。
  - centos7中默认是 ```/usr/share/nginx/html```， 可在Nginx conf中配置
```txt
npm install 
npm run build:prod --report
```	

## 解决问题 ##
### 跨域 ###
- [解决跨域问题](https://blog.csdn.net/qq_40369829/article/details/106275393)

## 参考 ##
- [vue文档](https://vuejs.bootcss.com/guide/)
- [vue菜鸟教程](https://www.runoob.com/vue2/vue-directory-structure.html)
- [vue+springboot实现登录](https://blog.csdn.net/zks_4826/article/details/81603865)
- [Element-UI](https://cloud.tencent.com/developer/doc/1270)
- [vuex文档](https://vuex.vuejs.org/zh/guide/)