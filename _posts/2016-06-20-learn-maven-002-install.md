---
layout: post
title:  Maven前世今生（2）—安装
date:   2016-06-20 23:04:00 +0800
categories: Maven
tag: 工具
---

* content
{:toc}

## 条件

Maven的安装使用，需要jvm和jre的支持，所以需要先确保本机有安装。jdk或jre的安装网上搜索能看到一大箩筐，所以这儿一笔带过。

## 下载

既然Maven是一个工具，也是一个软件，基础仍然是一个包。在官网就可以下载下来了。
https://maven.apache.org/download.cgi

## windows上的安装

windows上，是需要引入两个环境变量。
1. $M2_HOME，其指向的是maven包的解压缩路径
1. $PATH，加入$M2_HOME/bin

然后在命令行中输入“mvn --version”，查看返回信息。

## unix系的安装

如mac、centos和ubuntu，都可以通过在线安装的。同时也支持zip或tar.gz包的解压缩安装。这儿推荐使用在线安装工具。至于如何安装，百度一下你就知道。

## settings.xml

这个文件在$M2_HOME/conf下有一份，我们可以将其拷贝到~/.m2下，然后修改。记住：用户级别的文件设置会覆盖全局级别的文件设置。在里门，我们设置一些有用的信息，如：本地仓库位置、镜像地址、发布仓库的用户信息等。

## 总结

此章写得比较简单，主要目的有两个。
1. 给自己备忘；
2. 给有一定java编程基础的用户查阅；