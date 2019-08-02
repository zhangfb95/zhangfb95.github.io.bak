---
layout: post
title:  sublime3常用插件集合
date:   2016-01-12 17:33:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}


* JSON插件Pretty JSON
Ctrl+Alt+J进行格式化

* Markdown Editing/Markdown Preview
可以编辑和预览markdown语法的文档

------

### 如何安装package control
参考文档 https://packagecontrol.io/installation
为了防止官网打不开的情况，现粘贴如下

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-c74d4192a72fc25f.png)

sublime3->
> import urllib.request,os,hashlib; h = '2915d1851351e5ee549c20394736b442' + '8bc59f460fa1548d1514676163dafc88'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)

sublime2->
> import urllib2,os,hashlib; h = '2915d1851351e5ee549c20394736b442' + '8bc59f460fa1548d1514676163dafc88'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); os.makedirs( ipp ) if not os.path.exists(ipp) else None; urllib2.install_opener( urllib2.build_opener( urllib2.ProxyHandler()) ); by = urllib2.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); open( os.path.join( ipp, pf), 'wb' ).write(by) if dh == h else None; print('Error validating download (got %s instead of %s), please try manual install' % (dh, h) if dh != h else 'Please restart Sublime Text to finish installation')

### 如何使用package control安装sublime3插件
1、使用快捷键【Ctrl+Shift+P】，在文本框中输入【install】，点击回车【Enter】。如下图所示：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-888d1f610f3d6f62.png)

2、在弹出框中输入插件名称，然后点击回车【Enter】进行安装。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-d2c492bcc098e6cf.png)

### 如何使用package control卸载sublime3插件

1、使用快捷键【Ctrl+Shift+P】，在文本框中输入【remove package】，点击回车【Enter】。如下图所示：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-b817a157f09b746a.png)

2、在弹出框中输入插件名称，然后点击回车【Enter】进行卸载。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-06ba0a7d3dca966c.png)

### 如何使用package control升级sublime3插件
类似安装、卸载，只是需要输入Upgrade Package。