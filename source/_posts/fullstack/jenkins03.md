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

node("master") {
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

## 打包
仅仅执行打包任务，也就是使用最新的源码构建安装包上传至samba服务器。
对应的commit为：`[package]`

## 打包并安装
先执行打包任务，然后将samba上面的安装包复制到安装节点执行自动化安装。
对应的commit为：`[reinstall]`

## 更新脚本文件
一些shell脚本没有包含在上面，这里我用一个新的提交注释叫`[update shell]`

## Jenkinsfile执行shell
在`Jinekinsfile`中通过`sh`执行shell脚本有很多现在，最好不要在里面写复杂逻辑，
可以将复杂逻辑拿出来写到一个单独的脚本文件中，比如我的打包和安装写到单独的`sh`文件中,
并且将它们纳入版本管理中。

下面是打包脚本：
``` sh
#!/bin/bash
# winstore package

param="$1"
cd /root/winstore_git/
if [[ "$param" -eq "winstore" ]]; then
    ./winstore_deploy.sh 4.0 >/dev/null
elif [[ "$param" -eq "all" ]]; then
    ./winstore_deploy.sh 4.0 mysql >/dev/null
fi
exit 0
```

下面是安装winstore集群脚本：
``` sh
#!/bin/bash
# winstore package

install_node="$1"
cluster_nodes_str="$2"
IFS=',' read -r -a cluster_nodes <<< "$cluster_nodes_str"

mount -t cifs -o username="samba",password="samba" //10.10.161.99/public /mnt/ -o rw
cd /mnt/winstore-liuhua/winstore4.0Beta/
ssh root@${install_node} "rm -rf /root/mysql-ansible" >/dev/null || true
ssh root@${install_node} "rm -rf /root/mysql_install*" >/dev/null || true
ssh root@${install_node} "rm -rf /root/winstore4.0_install*" >/dev/null || true
ssh root@${install_node} "rm -rf /root/winstore-ansible" >/dev/null || true
mysql_tar=`ls -l mysql_install* |tail -n1 |awk '{print $NF}'`
winstore_tar=`ls -l winstore4.0* |tail -n1 |awk '{print $NF}'`
echo "mysql_tar=${mysql_tar}, winstore_tar=${winstore_tar}"
scp ${mysql_tar} root@${install_node}:/root/ || true
scp ${winstore_tar} root@${install_node}:/root/ || true
cd /root/
umount /mnt/
ssh root@${install_node} "cd /root; tar zxf ${winstore_tar} &>/dev/null" || true
ssh root@${install_node} "echo '[localhost]' > /root/winstore-ansible/hosts" || true
ssh root@${install_node} "echo 'localhost ansible_connection=local' >> /root/winstore-ansible/hosts" || true
for i in "${cluster_nodes[@]}"; do
    if [[ "$i" != "$install_node" ]]; then
        echo "install node ip = ${i}"
        ssh root@${install_node} "echo \"$i ansible_connection=ssh ansible_user=root\" >> /root/winstore-ansible/hosts" || true
    fi
done

echo "start to install winstore"
ssh root@${install_node} "cd /root/winstore-ansible; ./install_winstore_simple.sh" || true
echo "end to install winstore"
exit 0
```

## 最后的Jenkinsfile

最终的`Jenkinsfile`文件如下：
```
#!groovy

node("master") {
    // 安装节点
    def install_node = "192.168.217.231"
    // 集群节点列表
    def cluster_nodes_str = "192.168.217.231,192.168.217.232,192.168.217.233"
    def cluster_nodes = ["192.168.217.231","192.168.217.232","192.168.217.233"]
    // 默认忽略所有push请求，除非commit说明包含特定字符串
    def skip = '-1'
    def workspace = pwd()
    stage('Check') {
        checkout scm
        // 一个优雅的退出pipeline的方法，这里可执行任意逻辑
        def resultUpdateshell = sh script: 'git log -1 --pretty=%B | grep "\\[update shell\\]" &>/dev/null', returnStatus: true
        def resultUpdateweb = sh script: 'git log -1 --pretty=%B | grep "\\[update web\\]" &>/dev/null', returnStatus: true
        def resultUpdateback = sh script: 'git log -1 --pretty=%B | grep "\\[update back\\]" &>/dev/null', returnStatus: true
        def resultUpdateall = sh script: 'git log -1 --pretty=%B | grep "\\[update all\\]" &>/dev/null', returnStatus: true
        def resultReinstall = sh script: 'git log -1 --pretty=%B | grep "\\[reinstall\\]" &>/dev/null', returnStatus: true
        def resultPackageWinstore = sh script: 'git log -1 --pretty=%B | grep "\\[package\\]" &>/dev/null', returnStatus: true
        def resultPackageAll = sh script: 'git log -1 --pretty=%B | grep "\\[package all\\]" &>/dev/null', returnStatus: true
        currentBuild.result = 'SUCCESS'
        if (resultUpdateshell == 0) {
            skip = '0'
            return
        }
        if (resultUpdateweb == 0) {
            skip = '1'
            return
        }
        if (resultUpdateback == 0) {
            skip = '2'
            return
        }
        if (resultUpdateall == 0) {
            skip = '3'
            return
        }
        if (resultPackageWinstore == 0) {
            skip = '4'
            return
        }
        if (resultReinstall == 0) {
            skip = '5'
            return
        }
        if (resultPackageAll == 0) {
            skip = '6'
            return
        }
        echo "Skipping ci build..."
        return
    }
    stage('Build') {
        if (skip == '-1') {
            echo "skipping building. lalala,"
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '0') {
            echo "Build Updateshell only... "
            for (node in cluster_nodes) {
                echo "Updateshell ${node}"
                sh """
                    cd resource/winstore/tools
                    sudo tar zcf configure.tar.gz configure/
                    sudo ssh root@${node} "rm -rf /opt/winstore/tools/configure/" 2>/dev/null || true
                    sudo scp configure.tar.gz root@"${node}":/opt/winstore/tools/ 2>/dev/null || true
                    sudo ssh root@${node} "cd /opt/winstore/tools/; tar zxf configure.tar.gz; chown -R sdsadmin:sdsadmin *; chmod +x configure/*; rm -f configure.tar.gz" 2>/dev/null || true

                    cd ${workspace}
                    sudo ssh root@${node} "rm -f /opt/winstore/venv/bin/query_wwid_disk.sh" 2>/dev/null || true
                    sudo scp update_resource/winstore/scripts/query_wwid_disk.sh root@"${node}":/opt/winstore/venv/bin/ 2>/dev/null || true
                    sudo ssh root@${node} "chown sdsadmin:sdsadmin /opt/winstore/venv/bin/query_wwid_disk.sh; chmod +x /opt/winstore/venv/bin/query_wwid_disk.sh" 2>/dev/null || true
                """
            }
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '1') {
            echo "Build Updateweb only... "
            sh '''
                echo "Updateweb ..........."
            '''
            for (node in cluster_nodes) {
                echo "Updateweb ${node}"
                sh """
                    cd resource/winstore/webapp/
                    sudo tar zcf winstore-web.tar.gz winstore-web/
                    sudo ssh root@${node} "rm -rf /opt/winstore/webapp/winstore-web/" 2>/dev/null || true
                    sudo scp winstore-web.tar.gz root@"${node}":/opt/winstore/webapp/ 2>/dev/null || true
                    sudo ssh root@${node} "cd /opt/winstore/webapp/; tar zxf winstore-web.tar.gz; chown -R sdsadmin:sdsadmin *; rm -f winstore-web.tar.gz" 2>/dev/null || true
                """
            }
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '2') {
            echo "Build Updateback only... "
            sh '''
                echo "Updateback ..........."
            '''
            for (node in cluster_nodes) {
                echo "Updateback ${node}"
                sh """
                    cd update_resource/winstore/
                    sudo tar zcf winstore.tar.gz winstore/
                    sudo ssh root@${node} "rm -rf /opt/winstore/venv/lib/python2.7/site-packages/winstore-3.6-py2.7.egg/winstore/" 2>/dev/null || true
                    sudo scp winstore.tar.gz root@"${node}":/opt/winstore/venv/lib/python2.7/site-packages/winstore-3.6-py2.7.egg/ 2>/dev/null || true
                    sudo ssh root@${node} "cd /opt/winstore/venv/lib/python2.7/site-packages/winstore-3.6-py2.7.egg/; tar zxf winstore.tar.gz; chown -R sdsadmin:sdsadmin *; rm -f winstore.tar.gz" 2>/dev/null || true
                    sudo ssh root@${node} "/etc/init.d/winstore-db restart; /etc/init.d/winstore-agent restart; /etc/init.d/winstore-operation restart; /etc/init.d/winstore-master restart; /etc/init.d/winstore-httpd restart" 2>/dev/null || true
                """
            }
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '3') {
            echo "Build Updateall only... "
            sh '''
                echo "Updateall ..........."
            '''
            for (node in cluster_nodes) {
                echo "Updateweb ${node}"
                sh """
                    cd resource/winstore/webapp/
                    sudo tar zcf winstore-web.tar.gz winstore-web/
                    sudo ssh root@${node} "rm -rf /opt/winstore/webapp/winstore-web/" 2>/dev/null || true
                    sudo scp winstore-web.tar.gz root@"${node}":/opt/winstore/webapp/ 2>/dev/null || true
                    sudo ssh root@${node} "cd /opt/winstore/webapp/; tar zxf winstore-web.tar.gz; chown -R sdsadmin:sdsadmin *; rm -f winstore-web.tar.gz" 2>/dev/null || true
                    cd ${workspace}
                """

                echo "Updateback ${node}"
                sh """
                    cd update_resource/winstore/
                    sudo tar zcf winstore.tar.gz winstore/
                    sudo ssh root@${node} "rm -rf /opt/winstore/venv/lib/python2.7/site-packages/winstore-3.6-py2.7.egg/winstore/" 2>/dev/null || true
                    sudo scp winstore.tar.gz root@"${node}":/opt/winstore/venv/lib/python2.7/site-packages/winstore-3.6-py2.7.egg/ 2>/dev/null || true
                    sudo ssh root@${node} "cd /opt/winstore/venv/lib/python2.7/site-packages/winstore-3.6-py2.7.egg/; tar zxf winstore.tar.gz; chown -R sdsadmin:sdsadmin *; rm -f winstore.tar.gz" 2>/dev/null || true
                    sudo ssh root@${node} "/etc/init.d/winstore-db restart; /etc/init.d/winstore-agent restart; /etc/init.d/winstore-operation restart; /etc/init.d/winstore-master restart; /etc/init.d/winstore-httpd restart" 2>/dev/null || true
                    cd ${workspace}
                """
            }
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '4') {
            echo "Build package only... "
            sh """
                echo "package winstore ..........."
                chmod +x resource/shell/jk_*.sh
                sudo resource/shell/jk_package.sh winstore || true
                echo "package winstore finished..."
            """
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '5') {
            echo "Build reinstall only... "
            sh """
                echo "reinstall ..........."
                chmod +x resource/shell/jk_*.sh
                sudo resource/shell/jk_package.sh all || true
                echo "package all finished..."
            """
            echo "Build reinstall successfully..."
            currentBuild.result = 'SUCCESS'
            return
        }
        if (skip == '6') {
            echo "Build package only... "
            sh """
                echo "package all ..........."
                chmod +x resource/shell/jk_*.sh
                sudo resource/shell/jk_package.sh all || true
                echo "package all finished..."
            """
            currentBuild.result = 'SUCCESS'
            return
        }
    }
    stage('Test') {
        currentBuild.result = 'SUCCESS'
        echo "Test do nothing..."
    }
    stage('Deploy') {
        if (skip == '5') {
            echo "step deploying..."
            sh """
                echo "Deploy reinstall ..........."
                sudo resource/shell/jk_install.sh "${install_node}" "${cluster_nodes_str}" || true
                echo "Deploy reinstall finished..."
            """
            currentBuild.result = 'SUCCESS'
            return
        }
        currentBuild.result = 'SUCCESS'
        echo "Deploy do nothing..."
    }
}
```

我测试下`[update shell]`，随便改点什么，在提交时候前面加上`[update shell]`

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins23.png)

点击构建查看详细构建日志：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins24.png)

成功，其他的情况我就一一演示了！

## 更进一步

这里我以一个python项目为例子来说明自动化的实现，经过对比后工作效率会比之前手动操作高了很多。
利用shell脚本、python脚本、git版本管理，最后在Jenkins这个平台上面就能实现常见的自动化工作。

这里我的任务都比较简单，并且就一个jenkins单节点，而且并没有涉及到繁重的任务，没用到agent节点，
权限和安全控制暂时还没做，以后会逐渐补全这些。

## FAQ
构建历史太多了咋办，写个脚本清空下：
``` groovy
def jobName = "winstore-pipeline"
def job = Jenkins.instance.getItem(jobName)
job.getBuilds().each { it.delete() }
// uncomment these lines to reset the build number to 1:
job.nextBuildNumber = 1
job.save()
println('clear succesfully...')
```
在系统管理->脚本命令行里面输入上面的脚本。点击运行即可

