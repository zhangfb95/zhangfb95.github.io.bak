---
layout: post
title:  ubuntu中成功配置vpn
date:   2016-02-19 16:07:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

年后我就从windows阵营转到linux阵营了。所以在pc上安装了ubuntu，但是一直未成功配置公司的vpn。今天在公司运维（特别感谢王跃东和徐波）的帮助下，终于配置成功了。现将配置过程记录下来。

我最开始使用openvpn命令行的方式来连接，提示： Initialization Sequence Completed，但是过一段时间就又提示连接超时，网上也没有找到很好的解决方案。

后面参考以下文章，加上运维同事的大力支持，终于配置上了。
http://ubuntuhandbook.org/index.php/2014/05/establish-openvpn-connection-ubuntu-1404/

1. 安装openvpn
```
sudo apt-get install network-manager-openvpn
```

2. 在系统网络图标-> VPN Connections -> Configure VPN

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-4dcec3e7ffb505e7.png?jianshufrom=true)

3.点击add按钮，在下拉框中选择OpenVPN

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-c28fdd3f8218987c.png?jianshufrom=true)

4.在配置页面添加正确的信息

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-87262e6a3eb7ca87.png?jianshufrom=true)

其中网关需要查看.ovpn配置文件中的配置；类型根据实际需求来选择，在现在公司使用的是密码方式；加入ca证书（以crt结尾的文件）。
**【注】：在高级中勾选tcp，否则vpn连接不上**

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-c8142c7838752c3d.png?jianshufrom=true)

5.所有配置完成之后，点击连接即可成功连接

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-8f530c798432e6fb.png?jianshufrom=true)

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/845143-d1ac0653d54c6634.png?jianshufrom=true)