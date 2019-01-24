---
title: 使用 Hexo 搭建自己的独立博客
date: 2019-01-15 23:38:11
categories: "Hexo" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Hexo
---

## Hexo 基本使用

### 依赖环境部署

首先需要安装 Node 和 Git。然后进入正题，安装 Hexo。

``` shell
sudo npm install -g hexo
```

### 初始化

终端cd到一个你选定的目录（比如创建一个blog文件，$cd blog），执行hexo init命令：

``` shell
hexo init
```

在blog目录下，执行如下命令，安装npm：

``` shell
npm install
```

执行如下命令，开启hexo服务器：

``` shell
hexo s
```

此时，浏览器中打开网址http://localhost:4000，能看到如下页面：

![hexo 初始页面](初始页面.png)

## Hexo + github.io 发布博客

## 创建 github 个人博客仓库

### 准备工作

在 github 创建一个以 “userId + github.io” 的 仓库，如我的 github 用户名是 gcyml，我创建了一个名为 gcyml.github.io 的仓库。

安装 Hexo 对应插件：

``` shell
npm install hexo-deployer-git --save
```

## 上传博客到 github

刚刚创建的 blog 目录中内容为：

```
_config.yml
db.json
node_modules
package.json
scaffolds
source
themes
```

切换blog文件夹下，使用文本编辑器打开 _config.yml，修改成下边的样子：

``` yml
deploy:
    type: git
    repository: https://github.com/gcyml/gcyml.github.io.git
    branch: master
```

需要注意把 gcyml 换成你自己的用户名

执行命令，生成静态页面，并上传到 github 上：

``` shell
hexo g -d
```

打开链接 http://gcyml.github.io 即可看到你的个人博客页面了。

## Hexo 引用自带图片

有两种方法可以实现，一种不用插件，需要 Hexo 版本达到 Hexo3 以上；第二种需要安装 Hexo 插件。

### 配置 _config.yml

两种方法都需要把 _config.yml 配置文件中的 post_asset_folder 项改为 true。

配置成功后，每次使用 `hexo new` 创建出的文章都会生成同名文件夹，把图片放到该文件夹中。

### 引用自带图片实现

#### 方法一：不带插件

```
{% asset_img 这是一个新的博客的图片.jpg 这是一个新的博客的图片的说明 %}
```

用此种方法，而不是以前的 `![]()` 方法，前提是你的 Hexo 的版本是hexo3 以上，到 package.json 里面看一下吧。如果不是 hexo3 以上的版本，那就只能用第二种方法了。

#### 安装 Hexo 插件

前面一种方法我觉得在 Markdown 嵌入标签破坏了一致性，我更倾向于第二种方法。

安装插件

```
npm install hexo-asset-image --save
```

把图片放到文章的同名文件夹中，按正常 mearkdown 引用图片即可。

```
我现在写了一个段落，并且想在这个段落的某一个地方![图片上传失败...(image.png)]引入一张图片
```

## 用 Hexo 插件备份博客

因为上传到 github 上面的是 hexo 生成的静态页面，而不是 markdown 源文件。如果要迁移和备份就会比较麻烦。

网上备份博客的方案多是自己创建一个源文件的分支。但是这种方法，一个是初始化要一个个添加相关配置和源文件，第二个是后续的备份更新也比较麻烦。这里推荐用插件备份的方式。

### 安装备份插件

hexo版本3.x.x

``` shell
npm install hexo-git-backup --save
```

hexo版本2.x.x

```
npm install hexo-git-backup@0.0.91 --save
```

### 配置_config.yml

_config.yml 新增

```
backup:
  type: git
  message: hexo blog source
  repository:  
    github: git@github.com:gcyml/gcyml.github.io.git,source #这里改成你自己的，分钟“source”也改成自己的，如果Github不存在会自动新建
```

### 备份到github

``` shell
hexo backup
```

备份成功后，可以在 github 上查看我们新建的备份分支。迁移博客时，只需要将分支上的数据覆盖到当前目录下即可

## 感谢

[基于hexo+github搭建一个独立博客](https://www.cnblogs.com/MuYunyun/p/5927491.html)

[Hexo发布博客引用自带图片的方法](https://www.jianshu.com/p/cf0628478a4e)

[Hexo 搭建个人博客教程 - 8 - blog备份工具hexo-git-backup - 2018](https://faceghost.com/article/866261)