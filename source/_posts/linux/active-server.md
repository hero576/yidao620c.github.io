---
layout: post
title: "5分钟搭建一个license服务器"
date: 2016-10-16 20:22:22 +0800
toc: true
categories: linux
tags: [license]
---

作为一个码农对于那些NO.1的开发工具爱不释手，它们对框架的支持、界面、插件都是那么的优秀，
大大加快了开发的速度以及开发的乐趣，酷炫的界面也能大大的装一个逼。

对于暂时经济不宽裕的同学，比较明智的选择是Google一台active服务器即可，
有能力的同学不妨尝试自行架设，这也就是本文的目的啦。

喝水不忘挖井人，在此向服务器软件的作者Lanyu表示衷心的感谢。<!--more-->

## 服务器
这个网上都有我就不列地址了，支持很多的操作系统平台，这里我选的是linux64位的。

接下来，介绍如何部署到Linux服务器上，首先将xxxxxxxxxServer_linux_amd64上传到任意目录，
我这里是 /root/work/ 目录，先将名字改了，太长了
``` bash
mv xxxxxxxxxServer_linux_amd64 xnServer
```

接下来 需要把它运行起来，先加一个可执行权限
``` bash
chmod +x xnServer
```

运行
``` bash
/root/work/xnServer -p 1008 -u love122 -prolongationPeriod 999999999 -l 127.0.0.1
```
默认运行会出现以下信息，则为成功(省略)。如果要后台运行，请使用nohup命令。

启动参数说明：

1. -l 指定绑定监听到哪个IP(私人用)
2. -u 用户名参数，当未设置-u参数，且计算机用户名为^[a-zA-Z0-9]+$时，使用计算机用户名作为idea用户名
3. -p 参数，用于指定监听的端口
4. -prolongationPeriod 指定过期时间参数

## 守护进程
也可以通过supervisor实现守护进程自启动。关于supervisor前面有专门的一篇讲解，有兴趣可以去看看。

编辑`/etc/supervisord.conf`，添加一个应用程序：
```
[program:xn-server]
user=root
command=/root/work/xnServer -p 1008 -u love122 -prolongationPeriod 999999999 -l 127.0.0.1
stdout_logfile=/var/log/supervisor/xnserver_out.log
stderr_logfile=/var/log/supervisor/xnserver_err.log
autostart=true
autorestart=true
startsecs=6
```

注：测试发现这个`-p 1008`不起作用，最后总是1017端口。

## nginx配置
接下来，将自己的域名采用nginx反向代理过来, 修改`/etc/nginx/nginx.conf`配置：
```
server {
    listen 80;
    server_name _;
    root /var/www/html/;

    location / {
        proxy_pass http://127.0.0.1:1017;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    access_log off; #access_log end
    error_log /dev/null; #error_log end
}
```

重启nginx `service nginx restart`

如果嫌nginx麻烦，也不需要配置，直接用ip地址加端口号也是一样：http://127.0.0.1:1017/

这样就大功告成了！
