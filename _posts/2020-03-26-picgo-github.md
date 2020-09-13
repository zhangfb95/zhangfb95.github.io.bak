---
layout: post
title:  使用PicGo+GitHub搭建图床
date:   2020-03-26 22:29:32 +0800
categories: 工具
tags: 工具
---

* content
{:toc}

## 前言

作为程序员，我们很享受使用markdown来书写文章的过程。但是一篇好的文章，往往是图文并茂的。文字还好说，我们可以自己码字。但是图片呢？往往依托于“图床”，图床又分为免费版和收费版两种。

收费的图床主要基于公有云来实现的。比如：腾讯云、七牛云、阿里云等。

而免费版的图床，可以考虑基于GitHub来实现。

楼主比较穷，所以想基于GitHub来搭建图床。而为了方便我们的书写，我们常常基于PicGo+操作系统自带的剪切板功能，来让我们的写作不会因为图片的操作而有所中断。

基于此，本文诞生。

## GitHub配置

第一步、在github上创建一个公有的仓库。

![创建公有仓库](https://upload-images.jianshu.io/upload_images/845143-53c51c826138acec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二步、创建token

![创建token的入口](https://upload-images.jianshu.io/upload_images/845143-4098c671a9807088.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Developer Setting](https://upload-images.jianshu.io/upload_images/845143-323f6cf9c4ac15c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Set Token](https://upload-images.jianshu.io/upload_images/845143-94484daf011540f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![设置token名称和返回](https://upload-images.jianshu.io/upload_images/845143-73c718ea5acf2cc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第三步、复制token，需要特别注意的是，token只会呈现一次，所以一定要复制。

![复制token](https://upload-images.jianshu.io/upload_images/845143-c561d2a7ed7392b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## PicGo下载

在官网下载PicGo。官网地址：https://github.com/Molunerfinn/PicGo/releases。

## PicGo使用

第一步、设置Github图床。

![设置Github图床](https://upload-images.jianshu.io/upload_images/845143-76d14f2a930717e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二步、设置快捷键，在Windows下建议设置为`Ctrl + Alt + C`。

![设置快捷键](https://upload-images.jianshu.io/upload_images/845143-01879f71fd86d1ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第三步、打开更新助手、开机自启、上传前重命名。

![打开参数](https://upload-images.jianshu.io/upload_images/845143-1f3361ea0550e4a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按下快捷键`Ctrl + Alt + C`，点击确定按钮，即可上传到github。同时，PicGo会将上传后的地址以markdown图片链接的方式存放到剪切板，我们只需要按下快捷键`Ctrl + V`即可使用。

## 参考文档

+ https://github.com/Molunerfinn/PicGo
+ https://www.zhihu.com/question/49522048
+ https://blog.csdn.net/yefcion/article/details/88412025