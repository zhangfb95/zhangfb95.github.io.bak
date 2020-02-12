---
layout: post
title:  Visual Studio Code插件（Todo+插件）
date:   2020-01-06 22:45:32 +0800
categories: vscode
tags: vscode
---

* content
{:toc}

## 前言

对于工作时间稍长的童靴来说，一个好的事项工具将会对我们的时间管理起到至关重要的效果。在工作过程中，我们往往会和各类人打交道，也会做很多事情，还可能会将很多事情分配给别人。这些事情可以拆分成一个个小的事项。而如何正确、高效地管理这些事项，往往是一门学问。

对于事项，楼主经常纠结于具体的工具。已有的一些工具包括：滴答清单、番茄土豆、Microsoft todo list、Tower，等等。我们不能说这些工具不好，但是这些事项往往是寄存在别的平台之上。一旦平台不可用，我们的事项也将因此而没法管理，从而影响我们的工作，乃至生活。

楼主在无意间看到Visual Studio Code的插件 -- Todo+，这个工具非常简单，也能满足我们基本的事项诉求。最重要的是，它是基于本地文本文件的，而本地文本文件又可以使用Git等版本管理软件很好地管理起来。本地磁盘或Git仓库，只要两个里面有一个不出问题，事项数据将不会被损坏。

参考[【官网说明 - readme.todo】](https://github.com/fabiospampinato/vscode-todo-plus/blob/master/resources/readme.todo)，楼主将这个插件的基本语法进行了翻译说明，以做备忘。

## 安装

![安装示意图](https://upload-images.jianshu.io/upload_images/845143-caf3c2e1c078d76a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 使用

第一步、打开todo文件。按下快捷键`Ctrl + Shift + P`，然后输入todo，回车`Todo: Open`，即可打开todo文件。

![打开todo文件](https://upload-images.jianshu.io/upload_images/845143-88ab76be6960a456.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二步、添加事项。在`Todo:`下，我们可以通过快捷键`Ctrl + Enter`添加一个待办事项，待办事项前面是复选框。输入框前面是两个空格。再按一下此快捷键，即可撤销添加。

![添加事项](https://upload-images.jianshu.io/upload_images/845143-3a47e04b619fdb1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第三步、开始事项，在事项上按下快捷键`Alt + S`，即可开始一个事项。再按一下此快捷键，即可撤销开始。

![开始事项](https://upload-images.jianshu.io/upload_images/845143-58369afe9434777f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第四步、完成事项，在事项上按下快捷键`Alt + D`，即可完成一个事项。再按一下此快捷键，即可撤销完成。

![完成事项](https://upload-images.jianshu.io/upload_images/845143-75e5fc2ca272c337.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第五步、取消事项，在事项上按下快捷键`Alt + C`，即可取消一个事项。再按一下此快捷键，即可撤销取消。

![取消事项](https://upload-images.jianshu.io/upload_images/845143-f5fcd1d410b214d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第六步、归档事项，在`Todo:`下点击右键，选中Archive，即可将已完成和已取消的事项进行归档。

![归档操作](https://upload-images.jianshu.io/upload_images/845143-4576d357ff282847.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![归档效果](https://upload-images.jianshu.io/upload_images/845143-d1a27ff85d21e63f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

本文通过一些简单的快捷键，让我们对事项的添加、开始、完成、取消和归档操作有了一个直观的了解。同时，每种操作都伴随着一些特殊颜色标识的呈现。

这些操作都非常简单，而简单的东西，往往意味着高效和易用。通过`Todo+`插件，老婆再也不用担心我的事项管理不到位了，\^\_\^。
