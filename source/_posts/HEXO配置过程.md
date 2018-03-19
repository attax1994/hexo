---
title: Hexo配置过程
date: 2018-03-19 14:06:19
categories: 
- tutorial
tags: 
- hexo
- tutorial
---

# Hexo配置过程
最早的博客是放在[CSDN](http://blog.csdn.net/lqy455949477)上的。后来发现了[Hexo](https://hexo.io)这个框架，将其与Git Page结合可以直接部署在Github上，确实是非常方便。于是花了几小时的时间过了一下官网的文档，并尝试了将其部署到这个Repo的上来。

## 配置环境
首先，使用HEXO需要具备一系列的开发环境：
- [Node.js、npm包管理工具](https://nodejs.org/)
- [Git](https://git-scm.com/)

安装完成后可以在termial检查node和npm是否存在。
```shell
$ node -v
$ npm -v
$ git --version
```
最后记得一定要有自己的[Github](https://github.com/)帐号，要不然部署到哪里去呢？

## 使用Hexo
### Hexo安装
在npm配置完成后，就可以进行Hexo-cli的安装
```shell
$ npm install -g hexo-cli
```

### 建立项目
安装完成后，就可以使用Hexo-cli来生成Hexo的项目
```shell
# 创建项目，<folder>替换为具体的文件夹名称
$ hexo init <folder>

# 进入该文件夹
$ cd <folder>

# 安装node_modules依赖
$ npm install
```
安装完成后，就可以使用Hexo来启动这个项目，进行测试
```shell
$ hexo server

# 或者使用简写形式
$ hexo s
```
如果运行成功，Hexo默认使用的是`localhost:4000`端口，直接进入该地址即可查看。
如果该端口已被占用，则需要更换一个端口来运行：
```shell
# 使用4001或者其他的未被占用的端口
$ hexo s -p 4001
```

### GitHub的准备工作
这里需要分为两种情况来讨论，具体关乎branch的差别。
#### 1.branch: master
如果要在master分支上部署网站的话，就**必须建立一个名称为`<yourUserName>.github.io`（将你的Github账户名替换这个`<yourUserName>`）的Repo**，然后在进行部署推送的时候，就可以使用这个Repo的master branch。

#### 2.branch: gh-pages
如果不想要将网站部署在master分支，在某一个Repo下建立gh-pages分支，部署的时候再部署到这个分支上（部署的过程将会在后面描述）。

branch准备完成以后，需要在Github的Repo中，选择Settings -> Github Pages选项，将Source选择到你使用的branch上面，就可以打开Github Pages功能了。
{% asset_img gitpage.png GitPages %}

### Hexo在Git上部署
##### 1.Git的配置
设置好Git的用户名和email（如果以前有设置过，可跳过）
```shell
$ git config --global user.name "yourname"
$ git config --global user.email "youremail"
```
##### 2.安装部署工具
**在项目路径下**，安装Hexo for Git的部署工具
```shell
$ npm install hexo-deployer-git --save
```
##### 3.修改配置文件`_config.yml`
配置`_config.yml`文件中的deploy项。（注意属性名称的冒号`:`后面要有一个空格）
type选择git
repo输入你github项目(即将被部署页面等内容的项目)的url
branch选择你需要的branch，比如master或者gh-pages
```yml
deploy:
  type: git
  repo: <url address of your github repository>
  branch: master
```
当然也可以根据[官方文档](https://hexo.io/zh-cn/docs/deployment.html)，去配置其他source的部署。
##### 4.Hexo部署
以上配置完成后，就可以着手正式的部署了。
在项目路径下，打开terminal输入以下命令
```shell
# 生成静态文件
$ hexo generate
# 部署
$ hexo deploy

# 简写形式，g表示generate，d表示deploy
$ hexo g -d
$ hexo d -g
```
这样就能将Hexo生成的静态页面，推送到远端的Github上。
如果部署在master分支上，访问`<yourUserName>.github.io`。
如果部署在gh-pages分支上，访问`<yourUserName>.github.io/<repository>/`。