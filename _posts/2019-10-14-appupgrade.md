---
layout: post
title:  关于APP升级的服务端解决方案
date:   2019-10-14 23:42:47 +0800
categories: 其他
tag: upgrade
---

* content
{:toc}

## 前言

现代社会，几乎所有的APP都离不开移动互联网的支持，包括：Android、iOS和Windows PC应用等。我们经常玩的多人在线联机游戏、在线金融理财产品、IT从业者编写代码的桌面程序，这些都是APP，也都需要互联网。它们并不是上线之后就不再维护和升级，而是随着时间的推移，开发商会持续地添加更多的新特性，解决可能存在的问题，亦或者是去掉不好的功能。

这中间，版本管理是必不可少的。缺少了版本管理，我们将一筹莫展。

那么，如何可控地、高效地和持续地对APP进行升级呢？

## 关键点因素

我们先从APP本身来分析。一个APP可能存在多个支持的终端。比如“王者荣耀”这个游戏，既可以支持Android用户使用，也支持iOS用户使用，还支持沙盒环境下的Windows系统上使用。

对于游戏开发商来说，他们可能会每隔两周或者一个月，不定时地对APP进行维护升级，增加更多好玩的内容。

同时，如果某个平台下，某个版本的APP存在严重bug（比如：无限制刷金币）。如果游戏开发商感知到这种情况，必定会对有问题的版本进行强制升级，以期升到某个稳定的版本。

另外，站在用户的角度上，用户可能对某个版本有特殊的嗜好而不愿意升级。这种情况下，我们也可以通过对可升级版本进行忽略的方式来跳过某个版本。

接下来，我们对上面的一系列场景进行归纳总结，大体上知道了APP升级会涉及到以下几个影响的因素：

1. 多终端支持
2. 随时间迭代
3. 强制升级
4. 用户体验性

通过对以上因素进行综合考虑，我们可以得到一个最基本的、行之有效的版本升级解决方案。

## 解决方案

> PS：在关键点因素中，我们提到了用户体验性。其实这种因素和具体终端有比较紧密的联系。因此，最好的解决办法就是：`在各个终端之上，将已忽略的版本添加到忽略缓存列表中，从而跳过某个版本的升级`。在此，我们不对这种因素进行更深入的说明。

#### 1. 输入层

在输入层，我们需要知道两个参数：

1. versionNo; // 当前版本号
2. appType; // 应用类型，比如Android、iOS、PC等

#### 2. 处理层

在处理层，我们需要得到几个关键的信息

1. 当前版本号是否最新
2. 当前版本是否需要强制升级
3. 可升级的版本信息

#### 3. 输出层

通过处理层的转储，我们来到了输出层。这儿，我们需要输出以下更详细的信息

1. latestVersion; // 是否最新版本
2. forceUpdate; // 是否强制更新
3. versionUpgradeInfo; // 可升级的版本信息
    1. url; // 文件上传路径
    2. fileName; // 文件名
    3. versionDescription; // 版本说明
    4. lastVersionNo; // 最新版本号
    5. lastVersionName; // 最新版本名称

#### 4. demo

整个的demo程序，使用Java实现的代码大致是这样的。

```java
// 输入信息
public class AppInfo {
    private Long versionNo; // 当前版本号
    private int appType; // 应用类型
}

// 输出信息
public class UpdateInfo {
    private boolean latestVersion; // 是否最新版本
    private boolean forceUpdate; // 是否强制更新
    private String url; // 文件上传路径
    private String fileName; // 文件名
    private String versionDescription; // 版本说明
    private Long lastVersionNo; // 最新版本号
    private String lastVersionName; // 最新版本名称
}

// 处理逻辑
public UpdateInfo fetchUpdateInfo(VersionItemRepository versionItemRepository, AppInfo appInfo)
            throws VersionException {
        // 获取并校验最新版本
        VersionItem lastVersionItem = versionItemRepository.getLatestVersion(appInfo.getAppType());
        if (lastVersionItem == null) {
            throw new VersionNotExistException("未查询到版本记录");
        }

        UpdateInfo updateInfo = new UpdateInfo();
        if (appInfo.getVersionNo() >= lastVersionItem.getVersionNo()) {
            // 如果app的版本【大于或等于】数据库里面的最新版本，则说明当前app的版本已是最新
            updateInfo.setLatestVersion(true);
        } else {
            // 当最新记录的强制更新版本号【大于或等于】当前版本号，则需要强制更新
            boolean forceUpdate = lastVersionItem.getForceUpdateVersionNo() != null &&
                    lastVersionItem.getForceUpdateVersionNo() >= appInfo.getVersionNo();
            updateInfo.setForceUpdate(forceUpdate);

            // 返回下载地址、文件名称、版本描述及强制更新版本号等信息
            updateInfo.setUrl(lastVersionItem.getFileUrl());
            updateInfo.setFileName(lastVersionItem.getFileName());
            updateInfo.setVersionDescription(lastVersionItem.getVersionDescription());
            updateInfo.setLastVersionNo(lastVersionItem.getVersionNo());
            updateInfo.setLastVersionName(lastVersionItem.getVersionName());
        }

        return updateInfo;
    }
```

## 总结

这种版本升级的解决方案，是我们真实线上环境的实现逻辑。通过这种方式，达到了我们想要的预期效果。
也就是保证了对以下元素的正确处理：

1. 多终端支持
2. 随时间迭代
3. 强制升级

当然，不同的应用场景也可能会有不同的特殊处理。如果读者有更多的问题，或者更好的解决方案，也欢迎留言。本文只是起到一个抛转引玉的作用。