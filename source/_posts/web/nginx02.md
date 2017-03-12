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
另外，反向代理还可以启用高级URL策略和管理技术，
从而使处于不同web服务器系统的web页面同时存在于同一个URL空间下。

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
        }

        ## 2.用户访问 http://ip:port/README.md，则反向代理到
        ##   https://github.com/.../README.md
        location /README.md {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass https://github.com/yidao620c/scrapy-cookbook/blob/master/README.md;
        }
    }
```

重启报错，因为我的nginx没有配置https

## 配置HTTPS

使用https保证机器上安装了openssl和openssl-devel
``` bash
yum -y install openssl*
```

颁发证书给自己
``` bash
openssl genrsa -des3 -out server.key 1024            \\ 用于生成rsa私钥文件
openssl req -new -key server.key -out server.csr     \\ openssl req 用于生成证书请求
openssl rsa -in server.key -out server_nopwd.key      \\利用openssl进行RSA为公钥加密
openssl x509 -req -days 365 -in server.csr -signkey server_nopwd.key -out server.crt
mv server.crt server_nopwd.key /usr/local/nginx/conf/
```

3，添加nginx证书
``` nginx
server {
    listen 443 ssl;
    ssl_certificate      server.crt; //证书路径，却对和相对路径都可以
    ssl_certificate_key  server_nopwd.key；
｝
```
保存退出

nginx -t检查出现
```
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf:99
```
原因也很简单，nginx缺少http_ssl_module模块，编译安装的时候带上`--with-http_ssl_module`配置就行了，
但是现在的情况是我的nginx已经安装过了，怎么添加模块，其实也很简单，往下看：

做个说明，我的nginx的安装目录是`/usr/local/nginx`这个目录，我的源码包在`/root/nginx-1.10.3`目录

切换到源码包`cd /root/nginx-1.10.3`，查看nginx原有的模块
``` bash
/usr/local/nginx/sbin/nginx -V
```
结果如下：
```
nginx version: nginx/1.10.3
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC)
configure arguments:
```
那么我们的新配置信息就应该这样写，在configure arguments基础上面加：
```
./configure --prefix=/usr/local/nginx --with-http_ssl_module
```
运行上面的命令即可，等配置完，运行命令:
``` bash
make
```
这里不要进行`make install`，否则就是覆盖安装。

然后备份原有已安装好的nginx
```
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```
然后将刚刚编译好的nginx覆盖掉原有的nginx（这个时候nginx要停止状态）:
```
systemctl stop nginx.service
cp ./objs/nginx /usr/local/nginx/sbin/
```
提示是否覆盖，输入y即可

然后启动nginx，仍可以通过命令查看是否已经加入成功
``` bash
/usr/local/nginx/sbin/nginx -V
```
结果:
```
nginx version: nginx/1.10.3
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC)
built with OpenSSL 1.0.1e-fips 11 Feb 2013
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx --with-http_ssl_module
```

重启 nginx 后，我们打开浏览器，验证下反向代理的效果。
在浏览器地址栏中输入`http://www.xiongneng.cc/README.md`，返回的结果是我的GitHub上面的README页面:

![](http://xnstatic-1253397658.cossh.myqcloud.com/nginx-09.png)

## 虚拟主机
有时候我们需要在同一台主机上面托管多个应用，每个应用访问域名不同，这里我们可以使用基于域名的虚拟主机来实现。

假设我们在本地开发有3个项目，分别在hosts里映射到本地的127.0.0.1上：
```
127.0.0.1 www.xiongneng.com xiongneng.com
127.0.0.1 api.xiongneng.com
127.0.0.1 admin.xiongneng.com
```

有这样3个项目，分别对应于web根目录下的3个文件夹，我们用域名对应文件夹名字，这样子好记：
```
/home/xiongneng/www/www.xiongneng.com/
/home/xiongneng/www/api.xiongneng.com/
/home/xiongneng/www/admin.xiongneng.com/
```

每个目录下都有一个index.php文件，内容就是都是简单的输入自己的域名

下面我们就来搭建这3个域名的虚拟主机，很显然，我们要新建3个server来完成。建议将对虚拟主机进行配置的内容写进另外一个文件，
然后通过include指令包含进来，这样更便于维护和管理。不会使得这个nginx.conf内容太多：
``` nginx
main
events {
    ....
}
http {
    ....
    include vhost/www.xiongneng.conf;
    include vhost/api.xiongneng.conf;
    include vhost/admin.xiongneng.conf;
    # 或者用 *.conf  包含
    # include vhost/*.conf
}
```

include：主模块指令，实现对配置文件所包含的文件的设定，可以减少主配置文件的复杂度。

既然每一个conf都是一个server，前面已经学习了一个完整的server写的了。下面就开始：
``` nginx
# www.xiongneng.conf
server {
    listen 80;
    server_name www.xiongneng.com xiongneng.com;

    root /home/xiongneng/www/www.xiongneng.com/;
    index index.html index.htm;

    access_log /usr/local/var/log/nginx/www.xiongneng.access.log main;
    error_log /usr/local/var/log/nginx/www.xiongneng.error.log error;

    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        include        fastcgi.conf;
    }
}
```
其他两个类似就省略了，这样3个很精简的虚拟域名就搭建好了。重启下nginx，
然后打开浏览器访问一下这3个域名，就能看到对应的域名内容了。


