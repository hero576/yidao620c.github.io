---
title: Docker每天学一点05 - Registry
date: 2017-04-07 12:07:09
toc: true
categories: docker
tags: docker
---

前面已经学会怎样构建镜像了，这一章介绍怎样分发镜像给多个Docker Host使用，
可以通过复制Dockerfile、将镜像上传至公共Registry、搭建私有Registry三种方式。

这里将后面两种方式，怎样使用公共Registry和搭建私有Registry。<!--more-->

## 镜像命名
其实就跟maven库管理一样，镜像名字由两部分组成：repository 和 tag：
```
[image name] = [repository]:[tag]
```

如果执行 docker build 时没有指定 tag，会使用默认值 latest。其效果相当于：

```
docker build -t ubuntu-with-vi:latest
```

在实际使用中，我们最好指定版本（也就是标签tag），比如`httpd:2.3`

我们可以通过 docker tag 命令方便地给镜像打 tag

```
docker tag myimage-v1.9.1 myimage:1
docker tag myimage-v1.9.1 myimage:1.9
docker tag myimage-v1.9.1 myimage:1.9.1
docker tag myimage-v1.9.1 myimage:latest
```

当我们发布2.0版本的时候，将latest换成2.0版本：

```
docker tag myimage-v2.0.0 myimage:2
docker tag myimage-v2.0.0 myimage:2.0
docker tag myimage-v2.0.0 myimage:2.0.0
docker tag myimage-v2.0.0 myimage:latest
```

## 使用公共 Registry

保存和分发镜像的最直接方法就是使用 Docker Hub。

Docker Hub 是 Docker 公司维护的公共 Registry。
用户可以将自己的镜像保存到 Docker Hub 免费的 repository 中。
如果不希望别人访问自己的镜像，也可以购买私有 repository。



