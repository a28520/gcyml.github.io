---
title: Ubuntu16.04 安装最新稳定版 Node.js 和 npm
date: 2018-05-21 17:03:23
categories: "Node.js" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Node.js
     - npm
---
## 安装 Node.js

Ubuntu16 环境下默认安装的 Node.js 版本是 v4.6.x 的，npm 会报 v4.6.x 的有bug，不支持。
如果要安装新版本，需要添加 ppa。输入下面两条命令即可安装 8 版本的 Node.js, 如果不想安装 8 版本，可修改其中的 setup_8.x 为其他版本。

``` shell
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt-get install -y nodejs
```

安装完成后可以检查所安装的 Node.js 版本

``` shell
nodejs -v
```

## 升级 npm

安装 NodeJs 成功后，我尝试用 npm 安装项目依赖时，却抛出了如下错误

``` shell
Error: write after end
```

谷歌了一下，找不出原因。一查 npm 版本号，果然还不是新版本。死马当活马医，更新到最新版

``` shell
npm install npm@latest -g
```

问题解决

## npm 更换国内源

npm 安装插件由于是国外服务器，速度慢，换成淘宝的源。

``` shell
npm config set registry https://registry.npm.taobao.org
```

修改后可以通过这个进行测试

``` shell
npm config get registry
```
