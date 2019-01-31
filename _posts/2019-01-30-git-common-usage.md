---
layout: post
title:  Git常用开发流程
date:   2019-01-30 23:23:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

## 前言

我们在[【Git常用命令备忘】](https://juconcurrent.com/2016/07/12/git-usage/)中，对Git的所有命令进行了汇总。但是，Git在项目中如何价值最大化地实践，我们却没有提及，更没有深入挖掘。楼主希望通过本文，将自己在实际项目中对Git的使用描述出来，使之能让读者对Git有更深入地理解。

> 【注】：本文侧重于实践和运用，不会涉及到底层原理和git的详细使用。若读者有这方面的诉求，请参考下面文章：1. [廖雪峰-Git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)；2. [gitscm官网-底层原理](https://git-scm.com/book/zh/v1/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86)

## 名词解释

1. 资源库，又叫远程仓库，用于存放包括源代码和配置在内的项目文件
2. 工作副本，维护着版本管理系统，版本管理系统使用本地文件方式实现
3. 客户端工具，和资源库进行通信、同时维护工作副本版本的客户端工具
4. 项目，仓库存放着项目，项目是仓库的逻辑单元
5. 分支，项目的时间轴，项目可以有多条分支，不同分支允许有重叠的时间轴节点
6. 标签，可以简单地等同于项目版本，或者迭代里程碑

## 工作流程

楼主先根据自己的理解，给出工作流程的整体结构图~

![Git工作流程](https://upload-images.jianshu.io/upload_images/845143-3cf7b480c3827431.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了更清楚地说明工作流程，我们给出一定的解释说明

1. 【工作副本】可以通过`pull`操作从【资源库】拉取更新
2. 【工作副本】可以通过`push`操作往【资源库】推送更新
3. 【工作副本】通过【版本管理系统】（本地文件系统）维护着修改历史
4. 【工作副本】在`修改`之后，需要将修改结果`提交`到【版本管理系统】
5. 【工作副本】可以脱离【资源库】而独立地工作，【资源库】是为了【工作副本】可以`多人协作`而提供的管理工具

## 最佳实践

在Git的使用过程中，为了尽量避免或减少代码冲突，我们需要遵循一定的操作顺序。有时代码冲突不可避免，当出现了代码冲突，我们通过怎样的方式能够很快、很好地对冲突的代码进行合并操作呢？

1. 修改代码之前，先从资源库pull
2. 修改之后，提交之前，从资源库pull时如果出现代码冲突，切记不要直接进行合并操作，而是需要在修改完成，且提交成功后再从资源库pull，然后解决掉冲突
3. 修改之后，提交之前，从资源库pull时若没有代码冲突，将直接pull成功
4. push代码之前，先从资源库pull，如有冲突，需要先解决掉冲突
5. 冲突的解决需要将本地修改和远程修改进行对比，然后提供调整之后的文件

### Git基本信息设置

设置全局用户名和邮箱

```
git config --global user.name author # 将用户名设为author
git config --global user.email author@corpmail.com # 将用户邮箱设为author@corpmail.com
```

设置项目用户名和邮箱

```
git config user.name nickname # 将用户名设为nickname
git config user.email nickname@gmail.com # 将用户邮箱设为nickname@gmail.com
```

### 工作副本的创建和资源库的克隆

1. 使用`git init`命令可在当前文件夹下创建一个【工作副本】
2. 使用`git clone ${remoteUrl}`可从资源库pull一个项目

### 分支管理策略

借用[【廖雪峰-分支管理策略】](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013758410364457b9e3d821f4244beb0fd69c61a185ae0000)的说明，我们简单将其拷贝到这儿。

在实际开发中，我们应该按照几个基本原则进行分支管理：

首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，再把dev分支合并到master上，在master分支发布1.0版本；

你和你的小伙伴们每个人都在dev分支上干活，每个人都有自己的分支，时不时地往dev分支上合并就可以了。

所以，团队合作的分支看起来就像这样：

![分支管理策略](https://upload-images.jianshu.io/upload_images/845143-a8f0bc0ad45017f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 标签管理

标签其实是一种特殊的分支，但是其意义却和分支有着显著的不同。一个标签用于标识一个版本的发布。这儿的版本可以是正式版本，可以是快照版本，也可以是内测、公测版本，等等。

### IDEA集成git

当我们对Git命令使用得非常熟悉之后，我们需要进一步提高我们的工作效率。怎么提高？我们往往通过IDE来操作其中集成的Git。IDEA作为当前最受Java开发者欢迎的IDE工具，我们需要关注其中Git相关的常用操作。

1. pull操作（快捷键：Ctrl + T）

![pull操作](https://upload-images.jianshu.io/upload_images/845143-530c0cbcdfc000f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. push操作（快捷键：Ctrl + Shift + K）

![push操作](https://upload-images.jianshu.io/upload_images/845143-866dc1e25adcda4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 添加操作（快捷键：Ctrl + Alt + A）

![添加操作](https://upload-images.jianshu.io/upload_images/845143-f1aed15542eef09a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4. 提交操作（快捷键：Ctrl + K）

![提交操作](https://upload-images.jianshu.io/upload_images/845143-46dd559ae6a133da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 分支管理

![分支管理](https://upload-images.jianshu.io/upload_images/845143-d547f34f353ee176.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. 回退操作（快捷键：Ctrl + Alt + Z）

![回退操作](https://upload-images.jianshu.io/upload_images/845143-924172fe90119a6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

7. 设置资源库

![设置资源库](https://upload-images.jianshu.io/upload_images/845143-e9185a8d48940de3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

8. 显示修改历史，可显示单个文件，也可以显示整个项目

![显示修改历史](https://upload-images.jianshu.io/upload_images/845143-0d20d8b3e91a82b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

本文根据楼主多年的Git使用经验总结而出，主要从下面几个部分对git进行了简单地总结。

1. 对Git的理解
2. Git的工作流程
3. Git的最佳运用
4. 分支管理
5. IDEA和Git集成使用

楼主希望通过这样的讲述让读者对Git有一个更深入地理解，也让自己对Git有一个比较直观地总结。

## 参考链接

+ https://git-scm.com/book/zh/v2
+ https://www.cnblogs.com/joyang/p/4922441.html