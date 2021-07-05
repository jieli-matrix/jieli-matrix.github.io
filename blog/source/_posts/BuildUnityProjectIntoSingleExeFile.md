---
title: Unity项目构建为单独exe文件
date: 2021-04-18 11:50:56
tags:
    - unity
    - SFX
categories:
    - 游戏开发
---

## SFX介绍

我们在下载、安装软件或许会有这样的经历：解压后，在安装目录内点击exe文件，运行软件(游戏)；一般软件还会贴心地帮你新建快捷方式到桌面，这样只需要点击软件图标就可以运行。

虽然平时仅运行exe文件，但心思细腻的人或许会发现，软件的安装目录有其他文件夹，如资源文件夹、日志文件夹等等。因此看上去仅仅运行exe文件，但是其依赖于安装目录下的其余文件。

<!--more-->

<center><img src ="https://i.loli.net/2021/04/18/Fhm9XxVPJnbl7TI.png" width=50%></center>
<center><font size=2>以Clash为例</font></center>

Unity构建出的游戏项目具有类似的文件夹结构，以《多米诺骨牌》项目结构为例

<center><img src="https://i.loli.net/2021/04/18/yXx5jTweWQ3YNlS.png" width=70%></center>
<center><font size=2>多米诺骨牌项目目录</font></center>

其中domino.exe是项目构建生成的windows平台文件，其运行需要依赖domino_Data中的数据文件。对于这个简单的物理模拟游戏，我们希望其能够构建为一个单独的exe文件，不再需要依赖额外的数据文件。

为解决上述问题，我们可以考虑将domino.exe与domino_Data打包到同一个归档文件；当我们赋予该归档文件可执行权限后，执行该文件，其能自动解压出依赖数据文件夹domino_Data，执行domino.exe，达到我们想要的效果。

这里就不得不提到SFX (SelF-eXtracting archives)技术，自解压归档技术。著名的解压缩软件winrar与7zip都支持SFX，以下是winrar关于SFX的简介


> An SFX (SelF-eXtracting) archive is an archive, merged with an executable module, which is used to extract files from the archive when executed. Thus no external program is necessary to extract the contents of an SFX archive, it is enough to execute it. Nevertheless WinRAR can work with SFX archives as with any other archives, so if you do not want to run a received SFX archive (for example, because of possible viruses), you may use WinRAR to view or extract its contents.
> SFX archives usually have .exe extension as any other executable file.

非常有趣好玩的一点是，我们装了新系统之后第N件事是安装winrar，然而软件安装需要进行解压操作，然而我们又没有解压软件，这一点如何实现呢？其实winrar的安装包就是SFX格式，安装包可在无解压缩软件的系统上自行解压，避免了“鸡生蛋 蛋生鸡”的哲学问题。

## 通过winrar的sfx功能构建unity项目为单个exe文件

接下来演示如何通过winrar的sfx功能将unity项目构建为单个exe文件  

- 1 新建归档rar
<center><img src = "https://i.loli.net/2021/04/18/BDrbltnQ5dvLPY8.png" width=40%></center>

- 2 将构建的exe文件与数据文件夹一同拖入rar
<center><img src = "https://i.loli.net/2021/04/18/toSqpLYgmx41iKI.png" width=40%></center>

- 3 单击该rar，进入SFX高级选项设置并指定可执行文件名
 <center><img src = "https://i.loli.net/2021/04/18/5uTLAObyrX1BUtn.png" width=40%></center>

- 4 依次指定运行模式（临时文件、静默模式）、更新(提取与覆盖)以及给设置图标(可选)
  <center><img src = "https://i.loli.net/2021/04/18/VbQwoilrnxX5Wmk.png" width=30%><img src="https://i.loli.net/2021/04/18/XvYam2CeUOldboQ.png" width=30%><img src="https://i.loli.net/2021/04/18/5EDHpfwx3n2aVU7.png" width=30%></center>
  
然后把生成的exe文件放到非构建目录下去玩一玩~

<center><img src="https://i.loli.net/2021/04/18/ItmQWrSR54LoCVP.png" width=60%></center>
<center><font size=2>我蛮喜欢这个企鹅的！</font></center>

<center><img src="https://i.loli.net/2021/04/18/JTNKxAofd6VnCBW.png" width=30%><img src="https://i.loli.net/2021/04/18/oPQjE3gpOs6J8x4.png" width=30%></center>
<center><font size=2>游戏页面，支持两种视角</font></center>

## 胡思乱想

最初只是为了打包unity文件，在折腾的过程中了解到了SFX技术，顺便思考了其来源（非常符合Linux下万物皆文件的思想）。紧接着我就产生了“邪恶”的念头：把恶意代码通过SFX打包到exe文件中，这样就可以进行恶意攻击呀！果不其然，我去查了下，发现还真有这么干的历史实例...所以从某些网站上下载软件时，最好还是不要立刻赋予管理员权限执行，而是先查看其文件结构，以及在沙盒运行~