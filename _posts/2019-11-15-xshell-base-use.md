---
layout: post
title:  深入使用Xshell
date:   2019-11-15 23:50:17 +0800
categories: XShell
tag: XShell,工具
---

* content
{:toc}

工欲善其事，必先利其器。器者，工具也。

## 前言

在百度百科上面，有关于Xshell的详细说明描述，大致如下。

> Xshell <sup>[1]</sup>  是一个强大的安全终端模拟软件，它支持SSH1, SSH2, 以及Microsoft Windows 平台的TELNET 协议。Xshell 通过互联网到远程主机的安全连接以及它创新性的设计和特色帮助用户在复杂的网络环境中享受他们的工作。<br/>
Xshell可以在Windows界面下用来访问远端不同系统下的服务器，从而比较好的达到远程控制终端的目的。除此之外，其还有丰富的外观配色方案以及样式选择。

> 【注】：当前Xshell的最新版本为6.x。为了与时代接轨，楼主在本文中对Xshell的所有操作都将基于这个版本。如果读者发现有些功能不一致或不存在，请升级到此版本后再试。

闲话少说，让我们进入Xshell的正式体验环节吧。

## 安装Xshell

按照安装向导，我们很容易就可以安装好Xshell。这儿只需要注意一点，就是`目的地文件夹`最好别选择系统默认的文件夹，而是放在一个统一的文件夹内，比如：D:/software。

![Xshell安装图](https://upload-images.jianshu.io/upload_images/845143-619a9cc347eeb826.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时，为了让Xshell的价值最大化，我们还可以安装Xshell Plus，这里面就包含Xftp。

![Xshell Plus安装图](https://upload-images.jianshu.io/upload_images/845143-67dbe0520956a6cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在首次启动的时候，我们需要设置一下Xshell用户数据存放的文件夹。

![Xshell设置数据文件夹](https://upload-images.jianshu.io/upload_images/845143-eac143ca65c9bc71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

安装好Xshell之后，我们进入它的工作桌面，看上去非常地简洁和清晰。

![Xshell工作桌面](https://upload-images.jianshu.io/upload_images/845143-2b071864f9d91b86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 入门级使用

通过【菜单栏】->【文件】->【新建】，我们可以在弹出框中，快速输入我们的远程主机的信息。

![远程主机信息的输入](https://upload-images.jianshu.io/upload_images/845143-bfa8813af828a57c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在用户身份验证中，身份验证的方法有很多种。这儿我们使用Password的方式。当然，Xshell支持的方式不局限于这一种，主要包括以下几种方式：

1. Password // 密码的方式
2. Public Key // 公钥的方式，可录入密码短语
3. Keyboard Interactive
4. GSSAPI
5. PKCS11

![使用密码的方式进行身份验证](https://upload-images.jianshu.io/upload_images/845143-1948544bba3a136b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击连接之后，首次进入的话，将弹出是否保存主机秘钥的提示框，这儿我们选择【接受并保存】

![SSH安全警告](https://upload-images.jianshu.io/upload_images/845143-2a3de9f90e5ad7dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，我们已完成了基本的远程主机连接操作。然后，我们就可以在上面执行我们需要执行的命令了。

![简单的sh命令交互](https://upload-images.jianshu.io/upload_images/845143-b1b8d418ef530627.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 整体的布局

![Xshell整体布局](https://upload-images.jianshu.io/upload_images/845143-d0dab959a24abd0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参考上图，我们可以看出，其大体包含以下部分：

1. 菜单栏
2. 标准按钮
3. 地址栏
4. 会话管理器
5. 工作区
6. 会话属性栏

我们可以通过点击【菜单栏】->【查看】->【工具栏】，或者右键点击空白区域的方式，从而选择性地关闭一些显示的组件。

![选择性关闭组件](https://upload-images.jianshu.io/upload_images/845143-4ee84c53fd460ae0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 【菜单栏】 - 【文件】

在【菜单栏】->【文件】中，我们可以做以下操作：

1. 创建会话
2. 打开会话
3. 断开会话
4. 另存会话
5. 导入和导出
6. 文件传输
7. 查看属性
8. 退出Xshell

![【菜单栏】 - 【文件】](https://upload-images.jianshu.io/upload_images/845143-a0ec33978925e718.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

会话的整个生命周期操作，我们都可以在这儿进行操作。除此之外，为了方便我们在不同的主机上进行会话信息的迁移，Xshell也提供了导入和导出功能。

如果我们想将当前主机的一些文件传输到远程主机，可以选择性地通过多种文件传输方式来操作。Xshell本身支持以下4种方式，其中的ZMODEM方式最优，也是Xshell默认的传输方式。关于这几种方式有何区别及联系，请参考[【Xmodem、Ymodem、Zmodem】](https://blog.csdn.net/xiaoluoshan/article/details/71773769)


1. ASCII
2. XMODEM
3. YMODEM
4. ZMODEM

![Xshell支持的文件传输方式](https://upload-images.jianshu.io/upload_images/845143-fea582e04a7736ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 【菜单栏】 - 【编辑】

在【编辑】中，我们可以对打开的远程主机中的工作桌面进行操作。

#### 1. 复制选中的内容

![复制选中的内容](https://upload-images.jianshu.io/upload_images/845143-1def4a64d6c04442.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 粘贴 && 粘贴选择内容

![粘贴 && 粘贴选择内容](https://upload-images.jianshu.io/upload_images/845143-2b36992e1d6834fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3. 选择

包括【全选】和【选择屏幕】，他们有以下区别：

1. 【全选】为当前工作桌面上的所有内容都会被选中，包括多页。
2. 而【选择屏幕】只是对当前工作桌面上的当前屏幕内容进行选中，只会有一页。

![选择](https://upload-images.jianshu.io/upload_images/845143-cc0b1a26cfe55311.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 4. 到文本编辑器

【到文本编辑器】和【选择】的功能有些类似，只是它多了`复制` + `打开文本编辑器` + `粘贴到文本编辑器`，仅此而已。

![image.png](https://upload-images.jianshu.io/upload_images/845143-c114756093db748e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 5. 查找

【查找】可对当前工作桌面进行查找，并高亮显示。

![image.png](https://upload-images.jianshu.io/upload_images/845143-1485eebf6cbb0492.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 6. 清屏及滚动缓冲区清除

【清屏】和【滚动缓冲区清除】是有区别的，初次使用时可能会对他们有一些误解。

1. 【清屏】只是把屏幕清除干净，而如果我们使用鼠标向上滑动的话，仍然能看到以前输入的内容。
2. 【滚动缓冲区清除】不仅把屏幕清除干净，而且如果我们使用鼠标向上滑动的话，不能看到以前输入的内容。

![image.png](https://upload-images.jianshu.io/upload_images/845143-43e367e53ea774d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 【菜单栏】 - 【查看】

在【菜单栏】的【查看】中，我们可以对可视区域的某些组件进行显示或隐藏。同时，我们还可以进行切换主题、全屏显示、屏幕锁定等操作。

![工具栏](https://upload-images.jianshu.io/upload_images/845143-3fa087a5fd7ad607.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![撰写栏](https://upload-images.jianshu.io/upload_images/845143-74464aa966fdc3fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![快速命令栏](https://upload-images.jianshu.io/upload_images/845143-fa2aecceed8ac22e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![主题切换](https://upload-images.jianshu.io/upload_images/845143-9a1ed7dc23442e43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![简单、透明、总在最前面](https://upload-images.jianshu.io/upload_images/845143-8c1dfd07733589ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![全屏&锁定屏幕](https://upload-images.jianshu.io/upload_images/845143-c69220035596547d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 【菜单栏】 - 【工具】

在【工具】中，最强大的功能就是，对多个打开的会话执行相同的命令。例如我们输入一个top命令，所有打开的会话都将执行这个命令。

![发送键输入到所有会话](https://upload-images.jianshu.io/upload_images/845143-b73a880dd9bef330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

除此之外，【工具】中还有另外一些令人心动的功能。

#### 1. 配色方案

根据个人喜好，每个人可选择自己喜欢的配色方案。

![配色方案](https://upload-images.jianshu.io/upload_images/845143-4c27b69a07c3da5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 查看&编辑快捷键

![按键对应入口](https://upload-images.jianshu.io/upload_images/845143-c42f954683bfb20f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![按键对应弹出框](https://upload-images.jianshu.io/upload_images/845143-5e57eec4e8ef312d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3. 更换语言及选项

更换语言就不用说了，就是可以切换界面呈现的语言。比如：英语、中文、日语等。

![更换语言&选项](https://upload-images.jianshu.io/upload_images/845143-6a79db92894c5b83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 【菜单栏】 - 【选项卡】

在选项卡中，我们可以对选项卡进行很多操作。一个选项卡我们可以理解成一个工作桌面。常见的选项卡操作有以下几种：

1. 新建选项卡
2. 在上、下、左、右等方向创建新选项卡
3. 设置选项卡排列顺序
4. 关闭选项卡（关闭当前选项卡、关闭非当前选项卡、关闭所有选项卡）
5. 重命名选项卡的标题
6. 给选项卡设置颜色

![选项卡](https://upload-images.jianshu.io/upload_images/845143-6e40ef4cae0a2770.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 【菜单栏】 - 【窗口】

【窗口】和【选项卡】有一些区别和联系。【窗口】更多的是一个独立的可视化单元。而【选项卡】之间却有所联系，他们是在一个窗口中显示的。

在【窗口】中，有一些常见的操作：

1. 新建窗口
2. 全部关闭（关闭当前窗口、关闭非当前窗口、关闭所有窗口）
3. 排列窗口

![窗口菜单](https://upload-images.jianshu.io/upload_images/845143-850879db7eb5ea65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 会话管理器

我们可以在会话管理器中对会话及文件夹进行操作。会话的常见操作包括：

1. 打开
2. 新窗口打开
3. 复制、剪切、粘贴
4. 删除、重命名、另存为
5. 查看属性

![新建会话或文件夹](https://upload-images.jianshu.io/upload_images/845143-4150b05c5beca338.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![会话的常见操作](https://upload-images.jianshu.io/upload_images/845143-5e9ff97968d3a74a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 工作桌面

在工作桌面上，我们点击鼠标右键，可以弹出一个下拉菜单。有很多便捷的操作都在这里面。

![工作桌面右键弹出框](https://upload-images.jianshu.io/upload_images/845143-c90b8b854c049585.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这些操作大体上可归纳为以下内容：

1. 文件传输
2. 日志操作
3. 复制和粘贴（对选中的文本）
4. 全选操作
5. 到文本编辑器
6. 查找
7. 粘贴本地ip地址
8. 粘贴远程ip地址
9. 清屏操作
10. 全屏操作
11. 发送键输入到所有会话

## 常用快捷键

对于一个工具使用的熟练程度，除了对所有功能都能理解之外，还需要熟悉其快捷键。如何熟悉呢？楼主觉得无外乎以下两种方式：

1. 多使用和实践（实践是最好的老师）
2. 死记硬背（君不见，熟读唐诗三百首，不会做事也会吟）

在Xshell中，所有的快捷键，可通过【菜单栏】->【工具】->【按键对应】来查看：

![按键对应](https://upload-images.jianshu.io/upload_images/845143-1551f9f4d4873d89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么，在Xshell中，又有哪些快捷键是常用的呢？楼主根据自己以往的经验梳理了一下，大概有以下这些：

|快捷键|功能|
|---|---|
|Ctrl+Insert|复制|
|Shift+Insert|粘贴|
|Ctrl+Shift+F|查找|
|Ctrl+Shift+L|清屏|
|Ctrl+Shift+A|清屏和滚动缓冲区清除|
|Ctrl+Shift+B|滚动缓冲区清除|
|Shift+Alt+N|新选项卡|
|Ctrl+Shift+F4|关闭当前选项卡|
|Alt+Enter|全屏|
|Alt+A|总在最前面|
|Alt+N|新建会话|
|Alt+O|打开会话|
|Alt+C|断开会话|
|Ctrl+Shift+R|重新连接会话|
|Ctrl+Tab|移动到下一个会话|
|Ctrl+Shift+Tab|移动到上一个会话|
|Alt+{number}|查看第几个会话|

## 总结

楼主通过这么一篇罗里吧嗦的文章，将实际中对Xshell的使用进行了一个全面的、系统的和彻底的总结，在后续的工作和生活之中，我们对远程主机的操作将更有信心。

同时，在总结和记录的过程中，我们也学会了如何更好地学习一个工具。楼主相信，工具是死的，人是活的，如何把工具用活，这就要看个人的本事了。