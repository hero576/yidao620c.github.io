---
title: "nginx笔记 - 反向代理"
date: 2017-01-09 20:16:16 +0800
comments: true
toc: true
categories: web
tags: [nginx]
---

反向代理（Reverse Proxy）方式是指用代理服务器来接受网络上的连接请求，然后将请求转发给内部网络上的服务器，
并将从服务器上得到的结果返回给请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。

举个例子，一个用户访问 `http://www.xiongneng.cc/readme`，但是`www.xiongneng.cc`上并不存在readme页面，
它是从另外一台服务器上取回来，然后作为自己的内容返回给用户。但是用户并不知情这个过程。对用户来说，
就像是直接从`www.xiongneng.cc`获取readme页面一样。
这里`www.xiongneng.cc`这个域名对应的服务器就设置了反向代理功能。<!--more-->

反向代理服务器，对于客户端而言它就像是原始服务器，并且客户端不需要进行任何特别的设置。
客户端向反向代理的命名空间中的内容发送普通请求，接着反向代理将判断向何处(原始服务器)转交请求，
并将获得的内容返回给客户端，就像这些内容原本就是它自己的一样。如下图所示：

![](http://xnstatic-1253397658.cossh.myqcloud.com/nginx-proxy.png)

## 应用场景

反向代理的典型用途是将防火墙后面的服务器提供给Internet用户访问，加强安全防护。
反向代理还可以为后端的多台服务器提供负载均衡，或为后端较慢的服务器提供缓冲服务。
另外，反向代理还可以启用高级URL策略和管理技术，从而使处于不同web服务器系统的web页面同时存在于同一个URL空间下。

## 操作实例
Nginx 的其中一个用途是做 HTTP 反向代理，下面简单介绍 Nginx 作为反向代理服务器的方法。

> 场景描述：访问服务器上的`README.md`文件`http://www.xiongneng.cc/README.md`，
> 服务器进行反向代理，从`https://github.com/yidao620c/scrapy-cookbook/blob/master/README.md`获取页面内容。

`nginx.conf`配置示例：

``` nginx
    ## 下面配置反向代理的参数
    server {
        listen    80;
        ## 1. 用户访问 http://ip:port，则反向代理到 https://github.com
        location / {
            proxy_pass  https://github.com;
            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;

        ## 2.用户访问 http://ip:port/README.md，则反向代理到
        ##   https://github.com/.../README.md
        location /README.md {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass https://github.com/yidao620c/scrapy-cookbook/blob/master/README.md;
        }
    }
```

重启 nginx 后，我们打开浏览器，验证下反向代理的效果。
在浏览器地址栏中输入`http://www.xiongneng.cc/README.md`，返回的结果是我的GitHub上面的README页面。

