---
layout: post
title:  Maven版本升级方案
date:   2019-08-16 23:08:30 +0800
categories: Maven
tag: Maven
---

* content
{:toc}

## 背景及原因

稍大一点的项目，通常会涉及到多个git管理的仓库。如果每个仓库使用maven管理其构件，那么，maven构件之间极有可能存在依赖，并且这些依赖可能跨多个git仓库。

随着迭代的演进，maven构件的依赖如果不进行有效地治理，项目将变的复杂和不可控。另外，maven构件的版本也至关重要，需要进行统一管理。

那么，如何合理、高效地对maven构件版本进行管理呢？其实，我们可以采取以下措施：

1. 每次迭代发版后，需要对所有git项目的构件进行版本递增
2. 构件的依赖版本，统一在bom中进行管控
3. 通过自动化管理的方式代替人肉的方式，进一步避免出错

## 真实案例的措施

假设有一个真实的产品，产品名为bingo。在bingo这个产品中，我们对maven版本采用以下有效的措施。

#### 第一、对外部依赖的二方库和三方库进行特殊处理

1. 统一通过`bingo-dependencies`进行依赖
2. 依赖的属性名称必须声明为bingo-dependencies.version，且指定当前版本号

大体结构如下所示：

```xml
<properties>
    <bingo-dependencies.version>1.3-SNAPSHOT</bingo-dependencies.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.xxcompany</groupId>
            <artifactId>bingo-dependencies</artifactId>
            <version>${bingo-dependencies.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
#### 第二、版本的升级统一使用Jenkins + shell脚本的方式

Jenkins Job的链接地址为：http://testjenkins.xxcompany.com/view/test/job/bingo-mvn-version

#### 第三、新增的jenkins bingo项目增强配置

我们在【Jenkins配置说明】中对此进行更详细的说明。

## Jenkins配置说明

第一步、增加参数化配置，用于外部传入待升级的版本号

![Jenkins的配置信息](https://upload-images.jianshu.io/upload_images/845143-2c9f49815c07b962.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二步、在Jenkins中增加shell脚本的执行，用于配置待升级的git工程

```sh
/opt/soft/shell/bingo-mvn-version.sh http://git.xxcompany.com/java/common.git ${VERSION} all
/opt/soft/shell/bingo-mvn-version.sh http://git.xxcompany.com/java/bingo-common.git ${VERSION} all

/opt/soft/shell/bingo-mvn-version.sh http://git.xxcompany.com/java/service-user.git ${VERSION} micserviceapi
/opt/soft/shell/bingo-mvn-version.sh http://git.xxcompany.com/java/service-article.git ${VERSION} micserviceapi
/opt/soft/shell/bingo-mvn-version.sh http://git.xxcompany.com/java/service-order.git ${VERSION} micserviceapi

/opt/soft/shell/bingo-mvn-version.sh http://git.xxcompany.com/java/web-bingo.git ${VERSION}
```

第三步、最核心的shell脚本内容如下所示：

```sh
#!/bin/bash

# $1 git地址
# $2 版本号
# $3 路径名称
#    1. 若不为空且不为all，则切换至对应目录并执行maven指令。
#    2. 不为空且为all，则在root根目录下执行deploy操作
#    3. 其他情况，不执行后续动作。

check_err() {
    # 获取参数个数
    args_num = $#
    # echo $args_num
    if [ $args_num -ne 2 ] && [ $args_num -ne 3 ]; then
        echo '''
        参数意义:
            参数1: git地址
            参数2: 版本号
            参数3: 路径名称，若不为空，则切换至对应目录并执行maven指令。默认不执行后续动作。如果参数为all，则在root根目录下执行deploy操作。
                1. 若不为空且不为all，则切换至对应目录并执行maven指令。
                2. 不为空且为all，则在root根目录下执行deploy操作
                3. 其他情况，不执行后续动作。
        试例:
            ./1.sh http://git.xxcompany.com/java/service-user.git 1.3-SNAPSHOT micserviceapi
        '''
    fi
}

# 检查有无语法错误
check_err

# git clone http://git.xxcompany.com/java/service-user.git
git clone $1

# cd service-user/
project_name=$(echo $1|awk -F'/' '{print $(NF)}'|awk -F'.' '{print $1}')
cd $project_name

git checkout develop

# mvn versions:set -DnewVersion=1.3-SNAPSHOT -DgenerateBackupPoms=false -U --settings /opt/soft/jenkins/settings.xml -Dmaven.test.skip=true
/usr/local/apache-maven-3.3.9/bin/mvn versions:set -DnewVersion=$2 -DgenerateBackupPoms=false -U --settings /opt/soft/jenkins/settings.xml -Dmaven.test.skip=true

/usr/local/apache-maven-3.3.9/bin/mvn versions:update-property -Dproperty=bingo-dependencies.version -DnewVersion=$2 -DallowSnapshots=true -DgenerateBackupPoms=false -U --settings /opt/soft/jenkins/settings.xml -Dmaven.test.skip=true

git add .
# git commit -m "version upgrade to 1.3-SNAPSHOT"
git commit -m "version upgrade to $2"

git push

/usr/local/apache-maven-3.3.9/bin/mvn clean install deploy -N -U --settings /opt/soft/jenkins/settings.xml -Dmaven.test.skip=true

# 判断路径名称是否为空，若不为空，则切换至对应目录并执行maven指令。默认不执行后续动作
if [ -n "$3" ]; then
    if [ $3 = "all" ];then
        #echo "not empty string"
        /usr/local/apache-maven-3.3.9/bin/mvn clean install deploy -U --settings /opt/soft/jenkins/settings.xml -Dmaven.test.skip=true
    else
        cd $3
        /usr/local/apache-maven-3.3.9/bin/mvn clean install deploy -U --settings /opt/soft/jenkins/settings.xml -Dmaven.test.skip=true
    fi
fi

cd .. && rm -rf $project_name
```

## 总结

在工作的过程中，我们希望Maven版本的升级能做到自动化，通过和运维同事的共同协作，我们终于实现并达到了这种效果。因此，我们通过这篇文章来记录这个过程。
