---
title: Git的ssh配置
date: 2021-07-05 19:28:16
tags: 
    - Git
    - SSH
categories:
    - 技术
---

近期配置新电脑时，按照惯常的ssh免密配置步骤对git进行免密配置，然而在git push origin main时一直出现如下报错

``` shell
Permissiondenied (publickey).
fatal:Could not read from remote repository.
Pleasemake sure you have the correct access rights and the repository exists.
```
<!--more-->

因此在这篇文章中记录问题的解决过程（看官方文档...），并回顾下ssh免密的机制~

## SSH概述

Secure Shell（安全外壳协议，简称SSH）是一种**加密**的网络传输协议，可在不安全的网络中为网络服务提供安全的传输环境，其最常见的用途是**远程登录系统**。本篇博客将对SSH连接过程与其在远程系统登录的应用进行介绍，重点说明**非对称加密**机制。

## SSH连接过程

SSH既然是网络传输协议，因而其连接过程必然会涉及到客户端与服务端；我们通过介绍客户端向服务端发起连接请求，二者互相传输数据的过程，对SSH的**非对称加密**过程进行介绍。

<center><img src="https://onekb.oss-cn-zhangjiakou.aliyuncs.com/50453185/a014b9b6-0e5d-4b9b-bbfd-938efb3b8a22.png"></center>
<center><font size=2><p>SSH连接过程 图源阿里云博客</p></font></center>

由上图分析可知，SSH连接分为三个阶段：
- 1 服务器准备阶段
服务器生成密钥对（*公钥*和*私钥*）
- 2 双方互发公钥
该阶段又被认为是非对称加密协商阶段：通过该阶段，客户端和服务端都有彼此的公钥，这为下一段传输信息的加密方法做了准备。
**Note:使用对方的公钥对己方的的传输信息进行加密，这就是**非对称加密**；而**对称加密**则是指用己方的密钥对己方的信息进行加密。
- 3 双方数据传输
该阶段，又分为两步：
    -  1 服务端使用客户端的公钥对传输信息进行加密，客户端使用自己的私钥对传输信息进行解密；
    -  2 客户端使用服务端对传输信息进行加密，服务端使用自己的私钥对传输信息进行解密。

显然，1和2过程为对偶过程，其共同点都是用对方的公钥对传输信息进行加密，使用自己的私钥对对传输信息进行解密，这是一个**非对称加密**过程。

*其实网络传输协议还有很多，比如TCP、UDP协议，以后有空了也会做一个比较的。*

## SSH的应用：免密登录

SSH加密传输具有安全性，因此其可以用于服务器的免密登录的身份校验过程。

<center><img src="https://onekb.oss-cn-zhangjiakou.aliyuncs.com/50453185/cc6d6bd0-b745-42e1-ba0a-c0d5dcc66564.png"></center>
<center><font size=2><p>SSH免密登录过程 图源阿里云博客</p></font></center>

SSH免密登录相比于SSH连接过程要简单很多，因而二者的目的不同：SSH免密登录是指客户端A通过服务端B的身份校验进行登录；而SSH连接则需要保持整个会话双方信息传输的加密性。
因此，SSH免密登录主要有如下两阶段：
- 1 客户端A将公钥信息提前写入服务端B(authorized keys)
- 2 服务端B生成一串随机数（作为校验信息），使用客户端A的公钥对随机数进行加密传输给客户端A，A使用自己的私钥进行解密，将*解密后的信息发给服务端B*，服务端B进行校验通过后则发送允许登录凭证，A顺利登录；校验失败，则尝试其他方式进行登录。

可以看到，这一过程中只有单向加密过程：服务端B向A发送加密信息，而A向B发送的*解密信息*则是以明文进行传输的（未加密）。

## SSH配置git免密

基于上述原理，我遵循传统的ssh免密登录方式，在本地生成一对密钥，并将该密钥添加到github上的SSH Keys(对应服务端的authorized keys文件夹)，然后在git push origin master时出现了如下权限错误

``` shell
Permissiondenied (publickey).
fatal:Could not read from remote repository.
Pleasemake sure you have the correct access rights and the repository exists.
```

通过阅读[GithubDocs](https://docs.github.com/cn/github/authenticating-to-github/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)，发现需要在客户端A启动ssh-agent，并通过ssh-add将密钥注册到ssh-agent中，之后才能正常连接。

这里自然是涉及到ssh-agent的作用：ssh-agent能够管理本机上的私钥文件，使用不同的密钥连接到不同的主机时，需要要手动指定对应的密钥，而ssh代理可以自动帮助我们选择对应的密钥进行认证。通过命令

``` shell
ssh -v -T git@github.com
```

打印ssh连接信息，可以发现ssh-agent对当前所有私钥文件进行尝试，并成功选择相应的私钥文件进行客户端信息解密。

当然这里我还有未解之谜：
我在~/.ssh/config文件中对登录信息进行了配置

``` shell
Host yuran.github.com
    HostName github.com
    IdentityFile ~/.ssh/yuran_rsa
    User yuran
```

理论上，这样的方式也同样指定了连接github时使用的私钥文件，为什么还会出现"Permission denied"的错误呢？同样的事，在我连接服务器不会出现。所以，如果我的配置没有出错的话，只能认为github是通过ssh-agent进行git用户身份认证的（因此要认真读官方文档！）

以及config文件能够做更为强大的事，比如ssh端口转发等等~

## 参考博客

[SSH服务相关介绍](https://help.aliyun.com/document_detail/141305.html) 
[SSH认证流程与规范](https://blog.csdn.net/gediseer/article/details/54584329)这篇文章也很棒，解答了我的未解之谜2：客户端发起登录请求是带着公钥的，服务器对该公钥在authorized_keys进行查询，如果查询失败则直接拒绝连接。

无关紧要的感慨：第一次了解到SSH概念，是在大三下时参加ASC培训时的课后实验，我在自己的电脑上配置了两个linux虚拟机，对“服务端”和“客户端”一脸懵逼...非常感谢[hopeful](https://github.com/hopeful0)对我的耐心，以及教我内网穿透以应用ssh端口转发。之后疫情在家，在lch的指导下用ssh端口转发把jupyter notebook开在学校服务器上，借用之前做的内网穿透转发到本地端口，才得以用上多卡gpu。本科毕业将近一年，班级同学大多断了联系，一路孤身、两点一线却又平凡的本科生活如一潭死水，但这些技术探索的往事，却时常在记忆中重现。非常感谢你们出现在我的生命中，在我真正自由而无用的青春时代。