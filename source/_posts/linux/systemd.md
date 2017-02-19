---
title: "centos7上systemd详解"
date: 2016-06-07 12:22:15 +0800
comments: true
toc: true
categories: linux
tags: [systemd]
---
CentOS 7继承了RHEL 7的新的特性，例如强大的systemd，
而systemd的使用也使得以往系统服务的/etc/init.d的启动脚本的方式就此改变，
也大幅提高了系统服务的运行效率。但服务的配置和以往也发生了极大的不同，同时变的简单而易用了许多。

CentOS 7的服务systemctl脚本存放在：/usr/lib/systemd/，有系统 system 和用户 user 之分，
即：`/usr/lib/systemd/system` 和 `/usr/lib/systemd/user` <!--more-->

## 服务配置
每一个服务以`.service`结尾，一般会分为3部分：[Unit]、[Service]和[Install]，就以nginx为例吧，具体内容如下：
```
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## 配置项说明
下面分别解释下着三部分的含义

*[Unit]*

* Description : 服务的简单描述
* Documentation ： 服务文档
* After= : 依赖，仅当依赖的服务启动之后再启动自定义的服务单元


*[Service]*

* Type : 启动类型simple、forking、oneshot、notify、dbus
    - Type=simple（默认值）：systemd认为该服务将立即启动，服务进程不会fork。如果该服务要启动其他服务，不要使用此类型启动，除非该服务是socket激活型
    - Type=forking：systemd认为当该服务进程fork，且父进程退出后服务启动成功。对于常规的守护进程（daemon），除非你确定此启动方式无法满足需求，
    使用此类型启动即可。使用此启动类型应同时指定`PIDFile=`，以便systemd能够跟踪服务的主进程。
    - Type=oneshot：这一选项适用于只执行一项任务、随后立即退出的服务。可能需要同时设置 `RemainAfterExit=yes` 使得 `systemd` 在服务进程退出之后仍然认为服务处于激活状态。
    - Type=notify：与 `Type=simple` 相同，但约定服务会在就绪后向 `systemd` 发送一个信号，这一通知的实现由 `libsystemd-daemon.so` 提供
    - Type=dbus：若以此方式启动，当指定的 BusName 出现在DBus系统总线上时，systemd认为服务就绪。
* PIDFile ： pid文件路径
* ExecStartPre ：启动前要做什么，上文中是测试配置文件 －t
* ExecStart：启动
* ExecReload：重载
* ExecStop：停止
* PrivateTmp：True表示给服务分配独立的临时空间

*[Install]*

* WantedBy：服务安装的用户模式，从字面上看，就是想要使用这个服务的有是谁？上文中使用的是：`multi-user.target` ，就是指想要使用这个服务的目录是多用户。

每一个.target实际上是链接到我们单位文件的集合,当我们执行
``` bash
systemctl enable nginx.service
```
就会在 `/etc/systemd/system/multi-user.target.wants/` 目录下新建一个 `/usr/lib/systemd/system/nginx.service` 文件的链接。

## 操作示例

下面是几个最常用的service操作:
``` bash
# 自启动
systemctl enable nginx.service
# 禁止自启动
systemctl disable nginx.service
# 重新加载
systemctl reload nginx.service
# 启动服务
systemctl start nginx.service
# 停止服务
systemctl stop nginx.service
# 重启服务
systemctl restart nginx.service
```

## 参考资料

* [create systemd unit files](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/sect-Managing_Services_with_systemd-Unit_Files.html)
* [NGINX systemd service file](https://www.nginx.com/resources/wiki/start/topics/examples/systemd/)

