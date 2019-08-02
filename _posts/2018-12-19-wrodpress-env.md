---
layout: post
title:  本地如何搭建WordPress博客
date:   2018-12-19 15:35:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

## 前言

wordpress是一个博客脚手架，能在很短的时间内快速搭建并看到实际的显示效果，很不错。虽然入门很简单，但是在搭建的过程中仍然会遇到一些问题，特记录此博客以做备忘。

## 下载wordpress

在[wordpress官网](https://wordpress.org/download/)下载最新版本的安装包，其是一个压缩包。

## 快速安装

参考[5分钟安装向导文章](https://codex.wordpress.org/Installing_WordPress#Famous_5-Minute_Installation)，按照以下步骤来执行

1. 下载安装压缩包
2. 创建wordpress使用到的mysql数据库
3. 拷贝wp-config-sample.php到wp-config.php（在压缩包根目录下），在其中修改数据库连接信息
4. 将wordpress解压缩的所有文件拷贝到web服务器，可以是apache服务器，也可以是ngnix服务器，等等。在mac上，默认安装的web服务器为apache，所在目录为`/Library/WebServer/Documents/`。
5. 【可选】启动web服务器，然后在浏览器中输入http://localhost/wordpress，即可访问

至此，快速安装就完毕了。

## 更多问题

默认的wordpress主题相对来说比较简单，功能也比较单一。有时为了让我们的网站更漂亮，更能体现自己的个性，需要安装一些主题。不过我在wordpress安装主题的时候，出现了一些问题。

### 1. 本地测试wordpress安装主题需要FTP问题

解决参考link，https://jingyan.baidu.com/article/c275f6bac8a13ae33d7567d1.html。

打开wp-config.php，然后在此文件中添加下面三行代码。

```php
define("FS_METHOD", "direct");
define("FS_CHMOD_DIR", 0777);
define("FS_CHMOD_FILE", 0777);
```

![修改配置](https://upload-images.jianshu.io/upload_images/845143-beb81c662f07d248.png?jianshufrom=true)


### 2. 在1解决之后，出现了Wordpress安装主题：无法创建目录

解决参考link，https://jingyan.baidu.com/article/414eccf66e58dd6b431f0a2d.html。

关键问题在于目录权限的问题，wordpress所使用的user默认为`_www`，所以我们可以将整个wordpress目录权限变更为`_www`即可。

变更命令为：`sudo chown -R _www ./wordpress`

至此解决了主题安装的问题。