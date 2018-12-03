---
title: github+hexo搭建个人博客
date: 2015-04-13 14:39:32
categories:
reward: false
tags:
     - blog
---

![avatar](http://ooid5jqhw.bkt.clouddn.com/hexo.jpg)

## 前言
使用github创建的博客是属于静态网站博客，也就是把写好的文章生成HTML网页，然后上传到github网站，显示的也就是HTML网页，所以加载速度会很快。

## 安装
### 申请Github
#### 申请帐号
这里我们就不多讲了，小伙伴们可以点击[Github](http://www.github.com/)，进入官网进行注册。
#### 创建仓库
登录账号后，在Github页面的右上方选择New repository进行仓库的创建。
在仓库名字输入框中输入：

<!--more-->

```
你想要的名字.github.io
```
然后点击Create repository即可。

#### 生成添加秘钥
在终端（Terminal）输入：
```
ssh-keygen -t rsa -C "Github的注册邮箱地址"
```
一路enter过来就好，待秘钥生成完毕，会得到两个文件id_rsa和id_rsa.pub，用带格式的记事本打开id_rsa.pub，Ctrl + A复制里面的所有内容，然后进入[github-ssh](https://github.com/settings/ssh)将复制的内容粘贴到Key的输入框，随便写好Title里面的内容，点击Add SSH key按钮即可。

### 申请域名（非必须）
为了方便用其他域名访问blog，也可以使用默认的名字，如xxx.github.io也可以访问。

### 安装git
作用：把本地的hexo内容提交到github上去。
[Git官网](https://git-scm.com/downloads)下载相应平台的最新版本，一路安装即可。
[Git的下载与安装](Git的下载与安装)

### 安装nodejs
作用：用来生成静态页面的
[Node.js官网](https://nodejs.org/)下载相应平台的最新版本，一路安装即可。
[Node.js 安装配置](http://www.runoob.com/nodejs/nodejs-install-setup.html)

### 安装Hexo
#### 安装
``` bash
$ cd d:/hexo
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo g # 或者hexo generate
$ hexo s # 或者hexo server，可以在http://localhost:4000/ 查看
```

#### 修改网站相关信息
```
title: inerdstack
subtitle: the stack of it nerds
description: start from zero
author: inerdstack
language: zh-CN
timezone: Asia/Shanghai
```
注意：每一项的填写，其:后面都要保留一个空格，下同。

#### 配置统一资源定位符（个人域名）
```
url: http://cclk.cc
```

#### 部署配置
```
deploy:
  type: git
  repo: https://github.com/cclk/cclk.github.io.git
  branch: master
```

#### 部署
然后执行命令（为了hexo支持git）：
```
npm install hexo-deployer-git --save
```
每次部署的步骤，可按以下三步来进行。
```
hexo clean
hexo generate
hexo deploy
```
一些常用命令：
```
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo s -p 5000 #开启预览访问端口（端口5000）
hexo deploy #将.deploy目录部署到GitHub
hexo help # 查看帮助
hexo version #查看Hexo的版本
```

#### 一些基本路径
文章在source/_posts, 文章支持Markdown语法，可以使用一些MarkDown渲染工具。如果想修改头像可以直接在主题的_config.yml文件里面修改，友情链接，之类的都在这里。

### 安装Hexo主题-yilia
介绍下hexo的主题[yilia](https://github.com/litten/hexo-theme-yilia),风格布局都很好。
#### 安装
在hexo的工作目录
```
cd d:/hexo
git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
```
然后打开Hexo文件夹下面的_config.yml文件，修改里面的theme为yilia。
可以在themes/yilia内找到_config.yml文件，进行头像等定制。


### 绑定域名
1、在source文件夹中新建一个CNAME文件（无后缀名），然后用文本编辑器打开，在首行添加你的网站域名，如xxxx.com，注意前面没有 http:// ，(前面可以加www，就可以用www.xxx.com访问)，然后使用hexo g && hexo d上传部署。
2、在域名解析提供商
    （1）先添加一个CNAME，主机记录写@，后面记录值写上你的xxxx.github.io
    （2）再添加一个CNAME，主机记录写www，后面记录值也是xxxx.github.io
这样别人用www和不用www都能访问你的网站（其实www的方式，会先解析成 http://xxxx.github.io 然后根据CNAME再变成 http://xxx.com 即中间是经过一次转换的）。上面，我们用的是CNAME别名记录，也有人使用A记录，后面的记录值是写github page里面的ip地址，但有时候IP地址会更改，导致最后解析不正确，所以还是推荐用CNAME别名记录要好些，不建议用IP。
3、等十分钟左右，刷新浏览器，用你自己域名访问下试试

### npm速度慢的问题
可以使用国内的npm镜像，如[淘宝npm](https://npm.taobao.org/)

### hexo支持流程图

``` bash
cd blog
yarn add hexo-filter-mermaid-diagrams
```
具体见：[hexo-filter-mermaid-diagrams](https://github.com/webappdevelp/hexo-filter-mermaid-diagrams)