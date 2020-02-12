---
layout: post
title:  Git分支管理最佳实践
date:   2019-12-12 21:37:32 +0800
categories: Git
tags: Git
---

* content
{:toc}

## 前言

Git是一个优秀的版本控制软件，可以在无网络环境下对代码进行提交，在有网络环境下再将代码推送到远程仓库。同时，由于本地会存储一份代码提交历史，所以能有效避免中心化问题。

Git的使用也是非常简单的，我们可以通过命令来操作，也可以借助于图形化工具来进一步简化使用的复杂度。不过，要在实际项目中用好Git却需要花费一番心思的。尤其涉及到以下几点：

1. 如何避免功能之间的相互干扰
2. 如果快速对线上bug进行修复并上线
3. 如何保证功能提测和上线的一致性

楼主参考了网上的一些文章，也结合了项目的实际情况，在Git Flow这种最佳实践之上，对分支管理进行了一定调整改造，以期达到一个基本的能满足实际场景的Git分支管理最佳实践。

> PS: 本文绝大多数内容，都是转载自[【Git 在团队中的最佳实践--如何正确使用Git Flow】](https://www.cnblogs.com/wish123/p/9785101.html)

## Git Flow概述

就像代码需要代码规范一样，代码管理同样需要一个清晰的流程和规范。Vincent Driessen 大佬为了解决这个问题提出了 [A Successful Git Branching Model](http://nvie.com/posts/a-successful-git-branching-model/)。下面是Git Flow的流程图

![Git Flow流程图](https://upload-images.jianshu.io/upload_images/845143-9b3cca0f4eae8fdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的图你理解不了？ 没关系，这不是你的错，我觉得这张图本身有点问题，这张图应该左转90度，大家应该就能很容易理解了。

## Git Flow常用的分支

1. Production分支
也就是我们经常使用的Master分支，这个分支包含最近发布到生产环境的代码，最近发布的Release， 这个分支只能从其他分支合并，不能在这个分支直接修改

2. Develop分支
这个分支是我们的主开发分支，包含所有要发布到下一个Release的代码，这个主要合并于其他分支，比如Feature分支

3. Feature分支
这个分支主要是用来开发一个新的功能，一旦开发完成，我们合并回Develop分支，并进入下一个Release

4. Release分支
当你需要发布一个新Release的时候，我们基于Develop分支创建一个Release分支，完成Release后，我们合并到Master和Develop分支

5. Hotfix分支
当我们在Production发现新的Bug时候，我们需要创建一个Hotfix, 完成Hotfix后，我们合并回Master和Develop分支，所以Hotfix的改动会进入下一个Release

## Git Flow如何工作

#### 初始分支

所有在Master分支上的Commit应该对应到具体的Tag

![初始分支](https://upload-images.jianshu.io/upload_images/845143-97ccc6e966213d91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Feature分支

分支名 feature/*

Feature分支功能开发完成后，必须合并回Develop分支, 合并完分支后一般会删点这个Feature分支，但是我们也可以保留

![image](https://upload-images.jianshu.io/upload_images/845143-3503671d290d71c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Release分支

分支名 release/*

Release分支基于Develop分支创建，打完Release分之后，我们可以在这个Release分支上测试，修改Bug等。同时，其它开发人员可以基于开发新的Feature (记住：一旦打了Release分支之后不要从Develop分支上合并新的改动到Release分支)

发布Release分支时，合并Release到Master和Develop， 同时在Master分支上打个Tag记住Release版本号，然后可以删除Release分支了。

![image](https://upload-images.jianshu.io/upload_images/845143-08eeaf88229f3f77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 维护分支 Hotfix

分支名 hotfix/*

hotfix分支基于Master分支创建，开发完后需要合并回Master和Develop分支，同时在Master上打一个tag

![image](https://upload-images.jianshu.io/upload_images/845143-184f7c3a9dd71b46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Git flow工具

实际上，当你理解了上面的流程后，你完全不用使用工具，但是实际上对我们大部分人来说，很多命令就是记不住呀，流程就是记不住呀，怎么办呢？

总有聪明的人创造好的工具给大家用，那就是Git flow script。

#### 安装

+ OS X
brew install git-flow

+ Linux
apt-get install git-flow

+ Windows
wget -q -O - --no-check-certificate https://github.com/nvie/gitflow/raw/develop/contrib/gitflow-installer.sh | bash

#### 使用

*   **初始化:** git flow init

*   **开始新Feature:** git flow feature start MYFEATURE

*   **Publish一个Feature(也就是push到远程):** git flow feature publish MYFEATURE

*   **获取Publish的Feature:** git flow feature pull origin MYFEATURE

*   **完成一个Feature:** git flow feature finish MYFEATURE

*   **开始一个Release:** git flow release start RELEASE [BASE]

*   **Publish一个Release:** git flow release publish RELEASE
*   **发布Release:** git flow release finish RELEASE
    别忘了git push --tags

*   **开始一个Hotfix:** git flow hotfix start VERSION [BASENAME]

*   **发布一个Hotfix:** git flow hotfix finish VERSION

![image](https://upload-images.jianshu.io/upload_images/845143-028283f0fab500b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Git Flow GUI

上面讲了这么多，我知道还有人记不住，那么又有人做出了GUI 工具，你只需要点击下一步就行，工具帮你干这些事！！！

#### SourceTree

当你用Git-flow初始化后，基本上你只需要点击git flow菜单，然后选择`start feature`, `release`或者`hotfix`, 做完后再次选择git flow菜单，点击Done Action。我勒个去，我实在想不到还有比这更简单的了。

目前SourceTree支持Mac、Windows、Linux。

这么好的工具请问多少钱呢？ **免费！！！！**

![image](https://upload-images.jianshu.io/upload_images/845143-37cac14bf1c039c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/845143-02f9eb79c9fcaf89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考链接

+ https://www.cnblogs.com/wish123/p/9785101.html
+ https://blog.csdn.net/weixin_30902675/article/details/98193517
+ https://www.cnblogs.com/wolf-sun/p/9929910.html
