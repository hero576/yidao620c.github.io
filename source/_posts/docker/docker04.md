---
title: Docker每天学一点04 - Dockerfile
date: 2017-04-05 15:17:09
toc: true
categories: docker
tags: docker
---

是时候系统学习 Dockerfile 了，下面介绍 Dockerfile 中最常用的指令，完整列表和说明可参看官方文档。<!--more-->

## FROM

指定 base 镜像。

## MAINTAINER

设置镜像的作者，可以是任意字符串。

## COPY

将文件从 build context 复制到镜像。

COPY 支持两种形式：

1. COPY src dest
2. COPY ["src", "dest"]

注意：src 只能指定 build context 中的文件或目录。

## ADD

与 COPY 类似，从 build context 复制文件到镜像。
不同的是，如果 src 是归档文件（tar, zip, tgz, xz 等），文件会被自动解压到 dest。

## ENV

设置环境变量，环境变量可被后面的指令使用。例如：

```
ENV MY_VERSION 1.3
RUN apt-get install -y mypackage=$MY_VERSION
```

## EXPOSE

指定容器中的进程会监听某个端口，Docker 可以将该端口暴露出来。我们会在容器网络部分详细讨论。

## VOLUME

将文件或目录声明为 volume。

## WORKDIR

为后面的 RUN, CMD, ENTRYPOINT, ADD 或 COPY 指令设置镜像中的当前工作目录。

## RUN

在容器中运行指定的命令。

## CMD

容器启动时运行指定的命令。

Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效。CMD 可以被 docker run 之后的参数替换。

## ENTRYPOINT

设置容器启动时运行的命令。

Dockerfile 中可以有多个 ENTRYPOINT 指令，但只有最后一个生效。
CMD 或 docker run 之后的参数会被当做参数传递给 ENTRYPOINT。

下面我们来看一个较为全面的 Dockerfile：
```
# my centos dockerfile
FROM centos
MAINTAINER xn@enzhico.com
WORKDIR /work
RUN touch README.md
COPY ["README.md", "."]
ADD ["test.tar.gz", "."]
ENV WELCOME "welcome you, Jim"
```

然后执行步骤：

1. 构建前确保 build context 中存在需要的文件。
2. 依次执行 Dockerfile 指令，完成构建。

![](https://xnstatic-1253397658.file.myqcloud.com/docker15.png)

运行容器，验证镜像内容：

![](https://xnstatic-1253397658.file.myqcloud.com/docker16.png)

可以看到我的工作目录是/work，并且里面的内容也都正确复制和解压缩了。

进入容器，当前目录即为 WORKDIR。如果 WORKDIR 不存在，Docker 会自动为我们创建。

ENV 指令定义的环境变量在容器里面已经生效。

## RUN、CMD 和 ENTRYPOINT

RUN、CMD 和 ENTRYPOINT 这三个 Dockerfile 指令看上去很类似，很容易混淆。

简单的说：

1. RUN 执行命令并创建新的镜像层，RUN 经常用于安装软件包。
2. CMD 设置容器启动后默认执行的命令及其参数，但 CMD 能够被 docker run 后面跟的命令行参数替换。
3. ENTRYPOINT 配置容器启动时运行的命令。





