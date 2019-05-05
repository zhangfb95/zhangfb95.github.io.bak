---
layout: post
title:  properties文件乱码解决方案
date:   2019-05-05 22:31:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

## 前言

在使用Intellij IDEA作为集成开发环境的时候，如果使用properties作为配置文件，可能会因为文件编码不正确而出现乱码。为了尽量规避这种现象，我们可采取以下措施。

## 处理过程

#### 需要对每个IDEA项目，在Settings → Editor → File Encodings中做以下修改。

![image.png](https://upload-images.jianshu.io/upload_images/845143-b478657ae325684f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 已存在乱码的properties文件需要做以下修改

1.  假设乱码properties文件的名字为formal.properties
2.  创建临时properties文件，例如，可取名为a.properties
3.  Windows下，可使用记事本打开formal.properties文件，`Ctrl + C`拷贝到剪切板
4.  在IDEA中打开a.properties文件，`Ctrl + V`将复制的内容粘贴到里面
5.  删除formal.properties文件
6.  将a.properties重命名为formal.properties
7.  提交 && push代码

#### 已发布到Linux系统的properties文件，可通过以下修改方式进行调整

1.  使用`vim formal.properties`打开文件
2.  使用`set fileencoding`查看文件编码格式
3.  使用`set fileencoding=utf-8`设置文件编码格式
4.  复制已转换为ascii编码的内容，然后粘贴到formal.properties文件中

#### Linux系统下，修改Spring Boot生成jar包中的properties文件

1.  使用`vim *.jar`打开jar文件
2.  通过`/ keywords`，找到指定的properties文件
3.  回车，进入properties编辑页面
