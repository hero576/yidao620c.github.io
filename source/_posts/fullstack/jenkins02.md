---
layout: post
title: "Jenkins持续集成 - 管道详解"
date: 2017-03-22 09:32:16 +0800
toc: true
categories: fullstack
tags: [jenkins]
---
前面一篇介绍了Jenkins的入门安装和简单演示，这篇讲解最核心的Pipeline部分。

Jenkins Pipeline 就是一系列的插件集合，可通过组合它们来实现持续集成和交付的功能。
通过`Pipeline DSL`为我们提供了一个可扩展的工具集，将简单到复杂的逻辑通过代码实现。

通常，我们可以通过编写`Jenkinsfile`将管道代码化，并且纳入到版本管理系统中。比如：<!--more-->

```
// Declarative //
pipeline {
    agent any ①

    stages {
        stage('Build') { ②
            steps { ③
                sh 'make' ④
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' ⑤
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
            }
        }
    }
}

// Script //
node {
    stage('Build') {
        sh 'make'
    }
    stage('Test') {
        sh 'make check'
        junit 'reports/**/*.xml'
    }
    stage('Deploy') {
        sh 'make publish'
    }
}
```

* ① agent 指示Jenkins分配一个执行器和工作空间来执行下面的Pipeline
* ② stage 表示这个Pipeline的一个执行阶段
* ③ steps 表示在这个stage中每一个步骤
* ④ sh 执行指定的命令
* ⑤ junit 是插件`junit[JUnit plugin]`提供的一个管道步骤，用来收集测试报告

## 管道名词

几个重要的名词，讲一下它们是什么意思：

* Step 一个简单的任务，比如执行一个sh脚本
* Stage 将你的命令组织成一个更高一层的逻辑单元
* Node 指定这些任务在哪执行

Stage和Step可以放到一个Node下面执行，不指定就默认在master节点上面执行。
另外Node和Step也能组合成一个Stage。

## 定义管道
有两种定义管道的方式，一种是通过Web UI来定义，一种是直接写`Jenkinsfile`。推荐后面一种，因为可以纳入版本管理系统。

### Web UI方式

这里先介绍第一种方式，通过Web UI，首先点击“新建”：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins07.png)

填写一个名字，然后选择`Pipeline`

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins08.png)

点击“OK”后，在`Script`中写一个简单的命令：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins09.png)

保存后，点击左侧的“立即构建”：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins10.png)

然后在“Build History”下面点击“#1”进入此次构建详情，再点击左侧的“Console Output”查看输出：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins11.png)

我们看到了打印出来的“hello world”说明成功运行。

上面的例子演示了通过Web UI创建的一个最基本的管道执行成功的案例。使用2个步骤：

```
// Script //
node { ①
    echo 'Hello World' ②
}
// Declarative not yet implemented //
```

* ① node 在Jenkins环境中分配一个执行器和工作空间
* ② echo 在控制台输出一个简单的字符串

### Jenkinsfile方式
上面通过Web UI方式只适用于非常简单的任务，而大型复杂的任务最好采用`Jenkinsfile`方式并纳入SCM管理。
这次我选择从SCM中的`Jenkinsfile`来定义管道。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins12.png)

我这里配置了一个git仓库位置，然后我在该项目根目录放一个`Jenkinsfile`，其实就是我上一篇里演示的。

### Poll SCM 触发器
选择`Build Trigger`为`Poll SCM`，定时检查是否有push操作，这里我设置每隔2分钟检查一次。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins13.png)

### Push触发器
这个触发器我更加推荐，因为是实时的，但是需要先配置gitlab的Webhook。

选择`Build when a change is pushed to GitLab. GitLab CI Service URL: http://192.168.217.161:8080/project/scm-example`

复制后面那个URL，然后登录gitlab项目打开项目配置`Web Hook`，如果没有配置SSL可以将证书检查取消：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins14.png)

增加后可点击下面的`测试`，上面显示：`钩子执行成功：HTTP 200`则表示没问题。

然后再来jenkins里面配置push触发器，还能选择你要过滤那些分支，比如我只响应master分支上面的push操作。可以这样：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins15.png)

push钩子的大致流程是这样的：

1. push代码，Gitlab触发hook，访问Jenkins提供的api
2. `Jenkins Branch Filter`系统判断自己需要处理的分支是否有改动，如果有开始构建
3. 运行构建脚本

然后我再来测试下，修改代码，提交后push到远程仓库中，看到效果正常触发了：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins16.png)

Tips: 每个`Jenkinsfile`文件都应该以`#!groovy`为开头第一行

## 使用Jenkinsfile
接下来详细介绍一下怎样编写`Jenkinsfile`来完成各种复杂的任务。

Pipeline支持两种形式，一种是`Declarative`管道，一个是`Scripted`管道。

一个`Jenkinsfile`就是一个文本文件，里面定义了`Jenkins Pipeline`。
将这个文本文件放到项目的根目录下面，纳入版本系统。

简单起见，我现在只介绍`Declarative Pipeline`

### 部署三阶段

一般我们的持续交付都有三个部分：Build、Test、Deploy，典型写法：

```
// Declarative //
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
                sh 'make'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
                /* `make check` returns non-zero on test failures,
                 *  using `true` to allow the Pipeline to continue nonetheless
                 */
                sh 'make check || true' ①
                junit '**/target/*.xml' ②
            }
        }
        stage('Deploy') {
            when {
                expression {
                    /*如果测试失败，状态为UNSTABLE*/
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' ①
                }
            }
            steps {
                echo 'Deploying..'
                sh 'make publish'
            }
        }
    }
}
```

### 环境变量

Jenkins定了很多内置的环境变量，可以通过`env`直接使用它们：
```
echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
```

设置环境变量：
```
// Declarative //
pipeline {
    agent any
    environment { ①
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment { ②
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

### 使用多个agent
```
// Declarative //
pipeline {
    agent none
    stages {
        stage('Build') {
            agent any
            steps {
                checkout scm
                sh 'make'
                stash includes: '**/target/*.jar', name: 'app' ①
            }
        }
        stage('Test on Linux') {
            agent { ②
                label 'linux'
            }
            steps {
                unstash 'app' ③
                sh 'make check'
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
        stage('Test on Windows') {
            agent {
                label 'windows'
            }
            steps {
                unstash 'app'
                bat 'make check' ④
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
    }
}
```

上面的例子，在任一台机器上面做Build操作，并通过`stash`命令保存文件，然后分别在两台agent机器上面做测试。
注意这里所有步骤都是串行执行的。

## Multibranch Pipeline

多分支管道可以让你在同一个项目中，对每个分支定义一个执行管道。Jenkins或自动发现、管理并执行包含`Jenkinsfile`文件的分支。

这个在前面一篇已经演示过怎样创建这样的Pipeline了，就不再多讲。


