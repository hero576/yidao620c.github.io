---
layout: post
title: "Jenkins持续集成 - 使用案例"
date: 2017-03-25 09:32:16 +0800
toc: true
categories: fullstack
tags: [jenkins]
---

这篇演示实际使用案例，我们组开发的项目名称叫分布式存储winstore，使用的python语言开发。
之前的开发流程为：用最新的安装包在服务器集群上面安装并初始化配置。配置好和服务器源码映射后进行相应开发，
结果没有问题后，提交至版本管理系统，然后去某台机器上面执行`winstore_install.sh`脚本，
发布最新的安装包至某个samba服务器。最后将这个新包复制到安装服务器上面进行安装。<!--more-->

这时候还要分两种情况，一种是不需要重新安装和配置的在线升级，只需覆盖相应文件重启服务器即可。
另一种是必须重新卸载再安装，要完整的执行整个安装配置流程。

现在想将整个过程通过Jenkins做自动化，并通过提交注释来控制是否在线升级或者重新安装。

## Scripted Pipeline
前面一篇文章我都是用的`Declared Pipeline`来说明，对于简单的任务没问题，
但是涉及到更加复杂的就需要使用`Scripted Pipeline`了，本章我将使用脚本管道来完成。

这里我先定义一个大致框图，分4个阶段：

* `Check`阶段将源码pull下来后，对最后一次`commit`的注释进行分析，如果有关键字才会进行后续步骤
* `Build`阶段依据不同的注释执行不通过的操作
* `Test`阶段暂时保留用作测试
* `Deploy`阶段依据不同的情况进行远程文件更新或重新安装，以及后续的处理

初始`Jekinsfile`如下：
```
#!groovy

node {
    // 默认忽略所有push请求，除非commit说明指定了[update]或[reinstall]
    def skip = '1'
    stage('Check') {
        checkout scm
        // 一个优雅的退出pipeline的方法，这里可执行任意逻辑
        def result1 = sh script: 'git log -1 --pretty=%B | grep "\\[update\\]"', returnStatus: true
        def result2 = sh script: 'git log -1 --pretty=%B | grep "\\[reinstall\\]"', returnStatus: true
        echo "check update = ${result1} , check reinstall = ${result2}"
        if (result1 == 0) {
            skip = '2'
            return
        }
        if (result2 == 0) {
            skip = '3'
            return
        }
        echo "Skipping ci build..."
        return
    }
    stage('Build') {
        if (skip == '1') {
            echo "skipping building. lalala,"
            return
        }
        if (skip == '2') {
            echo "Build update only... "
            sh '''
                echo "update ..........."
                echo "update ..........."
            '''
            return
        }
        if (skip == '3') {
            echo "Build reinstall only... "
            sh '''
                echo "reinstall ..........."
                echo "reinstall ..........."
            '''
            return
        }
        /* 不断重试直到成功，最多尝试3次 */
        /*
        * retry(3) {
        *     sh '/opt/zz.sh'
        * }
        * timeout(time: 3, unit: 'MINUTES') {
        *     sh '/opt/zz.sh'
        * }
        */
    }
    stage('Test') {
        if (skip == '1') {
            echo "skipping testing. lalala,"
            return
        }
        echo "step testing..."
    }
    stage('Deploy') {
        if (skip == '1') {
            echo "skipping deploying. lalala,"
            return
        }
        echo "step deploying..."
    }
}
```

## 前期准备
有几个工作需要预先准备好

jenkins主机上面，编辑visudo，将tomcat用户加入`sudo`组并且可免密码执行"sudo"。

测试集群：
```
192.168.217.231
192.168.217.232
192.168.217.233
```
`192.168.217.231`为安装机器，保证该主机可ssh无密码登录其他主机。
并将这些IP地址配置写到`Jenkinsfile`开头的配置中。

保证jenkins主机上面root用户无密码访问集群的所有节点，
```
sudo su root
ssh-copy-id root@192.168.217.231
ssh-copy-id root@192.168.217.232
ssh-copy-id root@192.168.217.233
```

将winstore项目版本管理由svn迁移至git，我自己构建了一个内部的gitlab服务器。
新建一个名字为`winstore-ansible`的项目，在项目根目录添加之前的`Jenkinsfile`，
同时将本地最新项目push上来。

## 创建一个job
创建一个`Pipeline`类型的job，同时设置它的git仓库地址，
然后在gitlab上面配置刚刚创建好的`winstore-ansible`项目，指定其`webhook`。

做一个实验，随便修改点什么，push到远程仓库，看看是否触发刚刚的构建动作，我的演示成功：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins22.png)

## 仅仅更新
仅仅更新就是对每台主机执行替换操作，
这里分替换前端web，还是替换后台代码，再或者是两个都修改了替换。

我用三种commit来表示这三种情况：

1. [update web]
2. [update back]
3. [update all]

## 重新安装
先执行打包任务，然后将samba上面的安装包复制到安装节点执行自动化安装。
对应的commit为：`[reinstall]`

## 更进一步

这里我以一个python项目为例子来说明自动化的实现，经过对比后工作效率会比之前手动操作高了很多。
利用shell脚本、python脚本、git版本管理，最后在Jenkins这个平台上面就能实现常见的自动化工作。

这里我的任务都比较简单，并且就一个jenkins单节点，而且并没有涉及到繁重的任务，没用到agent节点，
权限和安全控制暂时还没做，以后会逐渐补全这些。

