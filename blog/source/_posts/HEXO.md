---
title: Hexo+GitPages搭建
date: 2021-01-17 16:52:24
tags:
    - Hexo
    - Web
categories:
    - 技术
---
接上文，近日通过Hexo + GitPages第二次搭建个人主页的过程中，记录遇到的问题与解决方案。
本文主要的结构如下：
- 介绍Hexo搭建本地博客基本流程
- 介绍GitPages开通与本地博客远端部署
- 探索Hexo基本框架
相比于初入Hexo框架时的自己，更希望能够对Hexo框架本身有更深的认识。此外，由于更换机器（目前由win切到mac）的可能性依然存在，因此也将博客源码进行托管。

<!--more-->

## Hexo搭建本地博客

Hexo是一种简洁、易用、高效的博客框架，使用Markdown解析文章并生成为静态网页(html/css/javascript)。更具体而言，当你在博客根目录下运行hexo g时，其遍历source目录（存储markdown文档），建立索引，根据博客根目录下的config.yml文件与themes目录下的config.yml文件生成静态页面(html/css/javascript)到public文件夹下。

对我而言，Hexo框架最方便的地方在于，作为用户可以专注于内容创作本身（掌握markdown文档的书写），而无需考虑为生成美观页面（由Hexo丰富的theme提供）从零写起前端代码。所以，接下来一起看看如何迈出第一步：在本地搭建一个博客吧～

### Node.js安装及包管理

Node.js是一个基于Chrome JavaScript运行时建立的一个平台，通过JavaScript语言开发web服务端。Node.js通过npm进行包管理（解决包之间的依赖问题），Hexo依赖于Node.js（通过Node.js平台进行页面生成），可由npm进行安装。

Node.js的安装方式有两种：官网下载或通过系统包管理(mac的brew或ubuntu的apt)。

``` shell
brew install node
node -v
npm -v #安装node后自带npm包管理
```

但对于不同场景，Node.js的版本可能会导致兼容问题，因此建议安装n(或nvm)对Node.js的版本进行管理（有点套娃的意思hhh）。

``` shell
sudo npm cache clean -f
sudo npm install -g n
sudo n 12.14.1 #12.14.1为需要安装的node版本号
sudo npm install npm@latest -g #更新npm包管理 
```

### Hexo安装

在搭建好Node.js环境后，通过npm安装Hexo

``` shell
npm install hexo-cli -g
```

之后，新建文件夹site用于存储博客文件。在site根目录下，运行如下命令

``` shell
hexo init blog
```

hexo init初始化blog文件夹用于存储其生成的静态网站，接下来，我们进行本地博客的生成与部署：

``` shell
cd blog # 之后的hexo命令，其都在blog文件夹作用域下
hexo g # 静态网页生成(public文件夹生成)
hexo s # 部署到localhost:4000
```

此时，我们访问http://localhost:4000，即可看到默认博客。

## GitPages开通与博客部署

考虑到开通博客的需求“可随时/多设备访问的笔记本”，因此需要将本地博客推送到GitPages上。
GitPages提供静态网页托管服务，其开通流程如下：
- 登陆Github账号，使用UserName新建同名仓库:UserName.github.io(对于Gitee，则仓库名与UserName同名)
- 默认分支为main，获取其http clone
如成果创建，可进入仓库的setting/GitHub Pages看到如下提示信息

``` shell
 Your site is published at https://username.github.io/
```

此时，回到本地博客blog目录下，打开根目录下的config文件，写入如下配置即可：

``` shell
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
- type: git
  repo: https://github.com/UserName/UserName.github.io.git
  branch: main
- type: git
  repo: https://gitee.com/UserName/UserName.git
  branch: main
```

此时，在终端执行如下命令：

``` shell
hexo clean
hexo g
hexo d
```
如看到部署成功的提示信息，则已完成。

### 源码备份

一般而言，main分支用于存储静态网页(也即blog/public/文件夹内容)，并不存储site/文件夹内容。因此，为了避免因更换机器迁移博客，建议新建hexo分支，将本地源码push到hexo分支。git checkout hexo后即可对源码进行修改。

## 探索Hexo基本框架

### hexo init blog过程

初始化blog文件夹后，生成核心目录信息如下：
- config.yml
- node_modules/
- package.json
- scaffolds/
- source/
- themes/

#### config.yml

blog根目录下配置文件，主要包含站点信息、主题配置、插件文件配置、部署配置。
站点信息配置：
``` shell
# URL
## If your site is put in a subdirectory, set url as 'http://example.com/child' and root as '/child/'
url: https://username.github.io/
root: /
```
插件文件配置：
``` shell
# sitemap
sitemap:
	path: sitemap.xml
baidusitemap:
	path: baidusitemap.xml
```
部署配置：

``` shell
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
- type: git
  repo: https://github.com/UserName/UserName.github.io.git
  branch: main
- type: git
  repo: https://gitee.com/UserName/UserName.git
  branch: main
```
注：在本地安装插件后，如在部署端未能生效，可以通过在config.yml文件中添加如下字段解决：

``` shell
plugins:
  hexo-generator-feed
  hexo-generator-searchdb
  hexo-generator-sitemap
  hexo-generator-baidu-sitemap
```

#### node_modules

node_modules是安装node后用来存放用包管理工具下载安装的包的文件夹，一般情况下不需要我们自己手动修改。

#### package.json

json文件，以字典形式存储依赖包的版本信息。

#### scafflods

页面布局文件夹，其子目录包含三个文件：
- draft.md
- page.md
- post.md
分别代表draft, page和post三种页面样式。一般而言，如果在导航栏生成新页面，则应执行如下命令：
``` shell
hexo new page tags
```
默认情况下，生成post样式作为博文
``` shell
hexo new daily #hexo new post daily
```

创建的新文件，存储在source目录下。

#### source
资源文件夹存储md文件，source文件夹的子目录对于不同页面文件夹，默认文件夹为_post/文件夹，我们写作时只需在_post/文件夹内的相应md文件创作即可。

#### theme
主题文件夹，可克隆多种喜欢主题，之后在config文件中设置对应主题即可。

### hexo g过程

#### public

在这一步中，hexo遍历source目录（存储markdown文档），建立索引，根据博客根目录下的config.yml文件与themes目录下的config.yml文件生成静态页面(html/css/javascript)到public文件夹下，同时生成db.json文件。
对于GitPages，其仅需要public文件夹里的内容，因此在main分支上仅自动部署public文件夹内容。网站实际根目录即为public目录。

## 总结

本篇博客记录了自己在重新部署hexo+gitpages遇到的问题，作为备忘以备日后查看。更重要的是，在这次部署中，更深入地了解到web的相关机制，对于hexo框架有了基本的认识。

参考内容：

[彻底搞懂如何使用Hexo+GitHubPages搭建个人博客](https://juejin.cn/post/6844904131266609165) 优质长文，推荐阅读
[Hexo的原理是什么？](https://www.zhihu.com/question/51588481)知乎提问，第二个回答十分精炼
