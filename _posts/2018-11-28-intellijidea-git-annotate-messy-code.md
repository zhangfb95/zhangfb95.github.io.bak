---
layout: post
title:  Intellij Idea - git annotate功能显示中文乱码
date:   2018-11-28 09:48:00 +0800
categories: Intellij IDEA
tag: IntellijIdea
---

* content
{:toc}

在使用idea进行功能开发的时候，我们有时候需要使用git的annotate功能来查看代码变更日志。

下图为annotate入口位置。

![入口位置](https://upload-images.jianshu.io/upload_images/845143-faee27397ede4ab1.png)

当提交的author中包含中文的时候，会出现乱码显示的情况，对我们查看提交人信息很不友好，如下图所示。

![中文显示乱码](https://upload-images.jianshu.io/upload_images/845143-6689033692a96f74.png)

这是因为idea的字体本身不支持中文的原因。怎么处理呢？其实我们只需要更换为支持中文的字体即可。只需要将Font从`Fira Code`变更为`Menlo`就可以了，这样就不会出现乱码问题了，😀

![第一步.png](https://upload-images.jianshu.io/upload_images/845143-860bf2c75cfe33c6.png)

![第二步.png](https://upload-images.jianshu.io/upload_images/845143-0e2512e7598955b3.png)

## 参考链接

https://blog.csdn.net/yuanyi0501/article/details/79516954