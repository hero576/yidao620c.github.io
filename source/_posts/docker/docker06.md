---
title: Docker每天学一点06 - 运行容器
date: 2017-04-09 09:16:33
toc: true
categories: docker
tags: docker
---

这一篇学习容器的各种操作，容器的状态之间如何转换，以及实现容器的底层技术。

运行容器

```
docker run httpd
```

查看当前正在运行的容器

```
docker ps
docker container ls
```

返回结果：
```
[root@VM_22_2_centos ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
2d33b4ef85c4        registry:2          "/entrypoint.sh /e..."   7 hours ago         Up 7 hours          0.0.0.0:5000->5000/tcp   vibrant_mestorf
ef4e11f77711        my-image            "/bin/bash"              2 days ago          Up 2 days                                    cranky_yalow
3d5ea3e73344        httpd               "httpd-foreground"       4 days ago          Up 4 days           0.0.0.0:8020->80/tcp     boring_bardeen
```

加一个`-a`选项可把所有容器都列出来：<!--more-->
```
docker ps -a
```

让容器长期运行，可以通过执行一个长期运行的命令来保持容器的运行状态，然后使用`-d`选项让它以后台方式启动容器。

```
docker run -d httpd
```

`CONTAINER ID` 是容器的 “短ID”，前面启动容器时返回的是 “长ID”。短ID是长ID的前12个字符。

`NAMES` 字段显示容器的名字，在启动容器时可以通过 `--name` 参数显示地为容器命名，
如果不指定，docker 会自动为容器分配名字。

对于容器的后续操作，我们需要通过 “长ID”、“短ID” 或者 “名称” 来指定要操作的容器。比如下面停止一个容器：
```
docker stop 3d5ea3e73344
```

查看容器启动时候执行的所有命令，`docker history` 会显示镜像的构建历史，也就是 Dockerfile 的执行过程。
注意前面讲的容器是一个层次结构，从底部向上面一层一层构建的：
```
docker history httpd
```

## 进入容器方法

我们经常需要进到容器里去做一些工作，比如查看日志、调试、启动其他进程等。有两种方法进入容器：attach 和 exec。

attach 方式：
```
docker attach long_id
```

exec方式：
```
docker exec -it <container> bash|sh
```

attach 与 exec 主要区别如下:

1. attach 直接进入容器 启动命令 的终端，不会启动新的进程。
2. exec 则是在容器中打开新的终端，并且可以启动新的进程。
3. 如果想直接在终端中查看启动命令的输出，用 attach；其他情况使用 exec。

如果只是为了查看启动命令的输出，可以使用 docker logs 命令：
```
docker logs -f 3d5ea3e73344
```

`-f` 的作用与 `tail -f` 类似，能够持续打印输出。

## 运行容器最佳实践

按用途容器大致可分为两类：服务类容器和工具类的容器。

1\. 服务类容器以 daemon 的形式运行，对外提供服务。比如 web server，数据库等。
通过 -d 以后台方式启动这类容器是非常合适的。如果要排查问题，可以通过 `exec -it` 进入容器。

2\. 工具类容器通常给能我们提供一个临时的工作环境，通常以 `run -it` 方式运行，比如：

```
docker run -it busybox

/# wget www.baidu.com
/# exit
```

运行 busybox，`run -it` 的作用是在容器启动后就直接进入。我们这里通过 wget 验证了在容器中访问 internet 的能力。
执行 `exit` 退出终端，同时容器停止。

工具类容器多使用基础镜像，例如 busybox、debian、ubuntu 等。

## 容器常用操作

启动、停止、重启容器：
```
docker stop boring_bardeen
docker start boring_bardeen
docker restart boring_bardeen
```

容器可能会因某种错误而停止运行。对于服务类容器，我们通常希望在这种情况下容器能够自动重启。
启动容器时设置 --restart 就可以达到这个效果。

```
docker run -d --restart=always httpd
```

暂停容器：
```
docker pause boring_bardeen
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker20.png)

处于暂停状态的容器不会占用 CPU 资源，直到通过 docker unpause 恢复运行：
```
docker unpause boring_bardeen
```

![](https://xnstatic-1253397658.file.myqcloud.com/docker21.png)

删除容器

使用 docker 一段时间后，host 上可能会有大量已经退出了的容器，
状态为Exited，这些容器依然会占用 host 的文件系统资源，如果确认不会再重启此类容器，
可以通过 docker rm 删除：

```
docker ps -a |grep "Exited"
docker rm de97841fbc9e
```

docker rm 一次可以指定多个容器，如果希望批量删除所有已经退出的容器，可以执行如下命令：
```
docker rm -v $(docker ps -aq -f status=exited)
```

注意：`docker rm` 是删除容器，而 `docker rmi` 是删除镜像。

## 容器状态

1. `docker create` 创建的容器处于 Created 状态
2. `docker start` 将以后台方式启动容器，容器状态处于 Running 状态。
3. 实际上，`docker run` 命令是`docker create` 和 `docker start` 的组合



