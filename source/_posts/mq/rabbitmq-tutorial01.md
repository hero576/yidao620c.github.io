---
layout: post
title: "RabbitMQ简易教程 - 安装"
date: 2017-05-06 10:55:13 +0800
comments: true
toc: true
categories: mq
tags: rabbitmq
---

最近又开始捣鼓RabbitMQ了，一个超好用的队列中间件，官网教程更新，自己也将有用的东东记录下来。

测试环境：消息服务器CentOS7.2、客户端Python3.6.1

RabbitMQ是一个出色的消息代理中间件（Message Broker）：接受和转发消息。你可以将它看作是一个邮局，
你把自己的信件写上收件人地址，然后放到邮筒里面就不用管了，由邮局负责将这个信件送到目的地。<!--more-->

## 几个术语

* producer: 消息发送者
* queue: 就是RabbitMQ里面的邮筒，消息只能存储在队列里面
* consumer: 消息接受者
* exchange: 消息路由器，所有的消息发送都需要经过路由器

## 安装
这里选择最简单的yum安装方式，其他安装方式请参考官网<https://www.rabbitmq.com/install-rpm.html>

先安装erlang
``` bash
rpm -Uvh http://download.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm
yum install erlang
```

安装RabbitMQ
``` bash
rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.9/rabbitmq-server-3.6.9-1.el7.noarch.rpm
wget ftp://195.220.108.108/linux/centos/7.3.1611/os/x86_64/Packages/socat-1.7.2.2-5.el7.x86_64.rpm
yum localinstall -C -y --disablerepo=* *.rpm
```

启动RabbitMQ
``` bash
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
```

安装web管理界面
``` bash
rabbitmq-plugins enable rabbitmq_management
```

创建配置文件，`vi /etc/rabbitmq/rabbitmq.config`，写入下面内容，我在这里指定了端口号5673，
另外还要注意最后的一个.：
```
[
    {rabbit,
      [
        {loopback_users, []},
        {tcp_listeners, [5673]}
      ]
    }
].
```

重启：
``` bash
systemctl restart rabbitmq-server
```

修改guest的密码:
```
rabbitmqctl list_users
rabbitmqctl change_password guest guest
```

如果你还想创建其他管理员账号比如test/test：
```
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
```

现在可以通过`http://ip:15672`访问web管理界面了，使用`guest/guest`登录。

![](https://xnstatic-1253397658.file.myqcloud.com/rb01.png)
