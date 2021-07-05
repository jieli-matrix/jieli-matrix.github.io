---
title: Unity游戏开发（一）：unity安装 
date: 2021-04-14 22:08:31
tags:
    - unity
    - tutorial
categories:
    - 游戏开发
---

这学期选修了数字孪生系统，需要借助unity游戏开发实现强化学习的demo，因此在这里记录unity环境部署与学习体会 (研究生课程全靠自学，听课就是听个关键词)。笔者的本机环境为windows，参考文档为[unity官方文档](https://docs.unity3d.com/cn/2021.1/Manual/GettingStarted.html)，参考慕课为[unity游戏引擎开发基础](https://www.coursera.org/learn/unity-yinqing-youxi-kaifa/home/welcome)。


<!--more-->

## Unity概述

Unity是一款2D/3D游戏引擎，用于多平台游戏开发(主流操作系统windows Linux Mac IOS Android以及任天堂、Xbox等游戏机)。一般而言，游戏开发是一个大型工程，需要美工、游戏策划、程序员与制作人的协作。  

就个人编程体验而言，游戏开发可类比web开发：
- 游戏场景设计 <---> UI设计
- 游戏场景实现 <---> html + css
- 游戏脚本(物体动作控制，C#编程实现) <--> javascript脚本

当然，Unity在某些方面难度是高于Web开发的(虽然我也没玩过Vue开发...求W大佬们不喷)，我以为是需要一定物理学和光学常识：比如多个相机的摆放位置，视角切换，物体间的位置、角度与碰撞关系，让玩家感到身临其境并不容易。

<center><img src="https://unity.com/sites/default/files/styles/810_scale_width/public/2019-05/dynamicvehicles2%20%283%29.png?itok=uQ3g5btL" width=70%></center>
<center><font size=2>炫酷unity场景</font></center>

作为一个不玩游戏的宅女，我口胡下好游戏应有的体验：

- 精美的游戏页面（游戏场景设计）
- 逼真的物理系统支持（游戏脚本与场景结合）
- 绝佳音效（游戏脚本与场景结合）
- 启动时间短，资源消耗合理范围(GTA5哈哈哈)

如果有好游戏推荐，麻烦捎上我一起去网吧体验！

## Unity安装

Unity推出了[Unity Hub](https://unity3d.com/cn/get-unity/download)用于管理不同的Unity版本和项目，可以认为Unity Hub类似Anaconda，而Unity对应虚拟环境，彼此隔离方便Unity项目管理。
因此安装思路为，第一步下载[Unity Hub](https://unity3d.com/cn/get-unity/download)，在Hub中安装对应版本的Unity，在Project中导入对应的项目文件。

### UnityHub配置

由于Unity具有大量的场景资源、音效资源等等，非常占用存储，因此建议不将其路径安装在C盘。Unity Hub可更改Unity的存储路径

<center><img src="https://i.loli.net/2021/04/14/EWDXfLYFO9CMHUy.png" width=60%></center>

单击右上方的多边形，即可修改存储偏好(别问...问就是C盘爆红之后的大彻大悟)

### Unity版本管理

配置好UnityHub之后，选择合适的Unity版本进行安装，退回到UnityHub主页面，从Installs选项卡中通过Add安装对应Unity版本

<center><img src="https://i.loli.net/2021/04/14/inL6cVR2rZjWHQa.png" width=60%></center>
<center><font size=2>Unity Hub管理</font></center>

通过UnityHub安装Unity速度较慢，因此也可以通过官网下载[Unity](https://unity3d.com/cn/get-unity/download/archive)，然后通过LOCATE导入到UnityHub。


#### Locate Unity Editor注意事项

由于通过UnityHub安装Unity速度较慢，因此我一般是通过官网下载Unity，再通过Locate定位到对应版本的unity.exe添加新版本。需要强调的是，打开[Unity](https://unity3d.com/cn/get-unity/download/archive)官网后，一般有若干个下载选项：

<center><img src="https://i.loli.net/2021/04/18/V1tkLP5ovyhWOAN.png" width=70%></center>
<center><font size=2>Unity Editor下载页面</font></center>

尽管从理论上而言，仅需要安装unity editor(64位或32位)，但考虑到项目的多平台部署需要安装对应的platform build，因此应下载第一个unity安装程序(unity editor不包含其他可选包，因此无法在多平台进行构建)。

这一点很重要，因为我打算将项目在windows平台构建时，发现unity editor没有其他平台的构建方案（然后卸载该版本unity重新装了安装程序...）

当然，在Unityhub下安装则会默认通过特定版本的Unity安装程序进行安装，并询问你可扩展包勾选哪些，在这两种情形下都请**务必勾选构建平台所需支持(windows mac linux android)**

<center><img src="https://i.loli.net/2021/04/18/1AGymhMO95ZRBvC.png" width=50%></center>
<center><font size=2>Unity平台支持务必勾选</font></center>

Unity的安装就暂时介绍到这，下一节会以一个基本项目为例，介绍Unity的开发界面。

