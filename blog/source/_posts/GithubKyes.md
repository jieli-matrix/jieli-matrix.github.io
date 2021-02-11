---
title: 同一机器部署多个Github账号
date: 2021-01-16 16:27:20
tags:
    - Git
categories:
    - 技术
---

近期由于记录需要，因此开通新博客用于个人技术积累与查阅。由于使用Github Pages静态托管服务，在进行ssh免密设置时，添加本机默认公钥id_rsa.pub遇到Github服务报错如下：
> Key is already in use
<!-- more -->

## 问题分析


这一问题来源于Github的安全性机制考虑：

> 出于对id_rsa.pub对应的私钥id_rsa的泄露安全性考虑,一个ssh-key只能配在一个单独的github账号（也即，同一个id_rsa.pub不能同时添加在user1和user2的github账号里）。因为一旦id_rsa私钥文件泄露，则所有作了免密的github账号都将被破解。

更为细致的解释与批量Github账号管理，参考[stackoverflow](https://stackoverflow.com/questions/51221571/unable-to-add-ssh-key-in-github)的解答（注册machine user，作为协作者可批量添加到多个账号中）。此处仅对两个账号的情况进行讨论。


## 解决方案

在解决方案中，涉及三步：
- 生成相应公钥（本地）
- 将公钥部署到相应账号（远端）
- 配置不同账号作用域（本地）
  
### 生成相应公钥

``` shell
ssh-keygen -t rsa -C "youremail" -f "username_rsa"
```

ssh-keygen用于生成密钥，其中-t指定rsa算法（另一种不常用的算法为dsa），-C为注释用于指示不同账号，-f用于命名生成密钥文件名（避免覆盖原始机器id_rsa文件）。

将username.pub文件添加到相应的github账号中。

之后，需要在本地的~/.ssh目录下修改config文件，以添加账号配置信息（明确不同username使用的私钥文件）。

``` shell
Host siran.github.com
        HostName github.com
        IdentityFile ~/.ssh/siran_rsa
        User siran
Host yuran.github.com
        HostName github.com
        IdentityFile ~/.ssh/yuran_rsa
        User yuran        
```

其中，不同字段的含义如下：
``` md
Host：别名（用于区分不同账号信息）
HostName：域名
IdentityFile：私钥文件
User：用户名
```

在终端进行测试

``` shell
ssh -T git@siran.github.com
ssh -T git@yuran.github.com
```
如能正确返回，则说明已配置成功。

### 配置不同账号作用域

由于在同一机器上配置了两个Github账号，因此每次在push操作时需要明确是哪个账号发起。
在只有机器上只使用一个账号时，我们一般倾向于使用git config -global进行全局配置；当使用多个账号时，我们则需要取消全局配置，对于每个仓库进行局部配置：

取消全局配置

``` shell
git config --global --unset user.name yourname
git config --global --unset user.email youremail
```

进行局部配置

在每个仓库内 进行局部配置

``` shell
git config user.name yourname
git config user.email yourname
```

设定仓库作用域
``` shell
$ git remote rm origin
$ git remote add origin git@siran.github.com:siran/demo.git 
```

至此，已经完成相应仓库的用户配置。

### 关于windows系统的git用户管理

上述方案从windonws切换到mac管理git用户无问题，但当从mac切换到windows对git用户管理则失效。其根本原因在于windows在初期配置了用户A的git credential信息
> Control Panel\User Accounts\Credential Manager 

打开控制面板，进入用户信息，查看证书管理会发现已配置了github与gitee的证书信息
![cred.png](https://i.loli.net/2021/02/11/CkDdgmjqXTVf9J2.png)

此时删除该证书，重新使用http方式clone仓库到本地，在推送时会弹出用户信息验证，即可顺利推送。

通过查阅Git凭证存储的文档，Git通过凭证存储系统对用户信息进行管理，以下是常见选项：
- 默认所有都不缓存。
每一次连接都会询问你的用户名和密码。
- “cache”模式 
将凭证存放在内存中一段时间。 密码永远不会被存储在磁盘中，并且在15分钟后从内存中清除。
- “store”模式
凭证用明文的形式存放在磁盘中，并且永不过期。密码是以**明文**的方式存放。
- mac系统加密
“osxkeychain”模式，它会将凭证缓存到你系统用户的钥匙串中。这种方式将**加密**凭证存放在磁盘中，并且永不过期。
- windows
Git Credential Manager for Windows会将用户信息以证书形式存储到证书中，之后遇到Git身份验证都会最先从证书中读取已有用户信息。

因此，在windows下建议使用cache模式，以避免多账号切换时带来的不必要麻烦。

