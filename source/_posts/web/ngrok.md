---
title: "CentOS7搭建ngrok服务器"
date: 2017-12-29 09:48:19 +0800
comments: true
toc: true
categories: web
tags: [ngrok]
---

ngrok是一个反向代理，它能够让你本地的web服务或tcp服务通过公共的端口和外部建立一个安全的通道，
使得外网可以访问本地的计算机服务。也就是说，我们提供的服务（比如web站点）无需搭建在外部服务器，
只要通过ngrok把站点映射出去，别人即可直接访问到我们的服务。

有做过微信公众号开发的人，对它应该不陌生。因为用户跟微信公众号产生的交互行为，微信会把用户的相关信息推送到我们自己的服务器，
如果服务在本地，那微信当然无法推送给我们，这使得开发功能的时候调试相当麻烦。我们可以使用ngrok把本地站点映射出去，解决这个问题。

另外如果我们想把本地开发时候的系统临时给外网用户看，无需部署到服务器上面去就可以，非常方便。

ngrok是开源的，官网地址：<https://github.com/inconshreveable/ngrok>

下面，我们开始搭建ngrok服务。操作系统为CentOS 7.2 <!--more-->

## 准备工作

搭建ngrok服务需要有一个外网服务器及一个域名解析到外网服务器上，我已经有了一个`xncoding.com`域名，并且拥有一台腾讯云主机。

在腾讯云主机的域名解析处，配置2个A记录，比如我新建2个`ngrok.xncoding.com` 和 `*.ngrok.xncoding.com` 解析到vps服务器上。

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok01.png)

## 搭建ngrok服务

### 安装go语言环境

ngrok是基于go语言开发的，所以需要先安装go语言开发环境，CentOS可以使用yum安装：

```
yum install golang
```

### 安装git

默认的git版本太低了，需要升级到git2.5，具体步骤如下：
```
sudo yum remove git
sudo yum install epel-release
sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
sudo yum install git2u
```

`git --version`，返回 `git version 2.5.0`，安装成功。

### 下载ngrok源码

新建一个目录，并clone一份源码：
```
mkdir ~/go/src/github.com/inconshreveable
cd ~/go/src/github.com/inconshreveable
git clone https://github.com/inconshreveable/ngrok.git
export GOPATH=~/go/src/github.com/inconshreveable/ngrok
```

### 生成自签名证书

使用ngrok.com官方服务时，我们使用的是官方的SSL证书。自己建立ngrok服务，需要我们生成自己的证书，并提供携带该证书的ngrok客户端。

证书生成过程需要有自己的一个基础域名，比如我的就是`ngrok.xncoding.com`。

```
$ cd ngrok
$ openssl genrsa -out rootCA.key 2048
$ openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=ngrok.xncoding.com" -days 5000 -out rootCA.pem
$ openssl genrsa -out device.key 2048
$ openssl req -new -key device.key -subj "/CN=ngrok.xncoding.com" -out device.csr
$ openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000
```

执行完成以上命令后，在ngrok目录下，会新生成6个文件：
```
-rw-r--r-- 1 root root   1001 Dec 29 11:53 device.crt
-rw-r--r-- 1 root root    903 Dec 29 11:44 device.csr
-rw-r--r-- 1 root root   1675 Dec 29 11:44 device.key
-rw-r--r-- 1 root root   1679 Dec 29 11:44 rootCA.key
-rw-r--r-- 1 root root   1119 Dec 29 11:44 rootCA.pem
-rw-r--r-- 1 root root     17 Dec 29 11:53 rootCA.srl
```

我们在编译可执行文件之前，需要把生成的证书分别替换到 `assets/client/tls`和`assets/server/tls`中，
这两个目录分别存放着ngrok和ngrokd的默认证书。
```
$ cp rootCA.pem assets/client/tls/ngrokroot.crt
$ cp device.crt assets/server/tls/snakeoil.crt
$ cp device.key assets/server/tls/snakeoil.key
```

### 使用lets encrypt免费证书

如果想让浏览器不弹出提示，最好不要使用自签名证书，现在lets encrypt推出泛域名证书了，所以可以先申请个免费域名证书。

客户端用证书 ：

```
cd ngrok
cp /etc/letsencrypt/live/xncoding.com/chain.pem assets/client/tls/ngrokroot.crt
```

服务器端用证书：

```
cp /etc/letsencrypt/live/xncoding.com/cert.pem assets/server/tls/snakeoil.crt
cp /etc/letsencrypt/live/xncoding.com/privkey.pem assets/server/tls/snakeoil.key
```

### 编译ngrokd和ngrok

首先需要知道，ngrokd 为服务端的执行文件，ngrok为客户端的执行文件。

接下来我们来编译ngrokd，在ngrok目录下，执行如下命令：
```
$ make release-server
```

编译过程需要等待一会，因为需要通过git安装相关依赖包。如果提示没有权限，使用 sudo 命令来安装。

由于客户端的平台版本较多，我们需要交叉编译来选择生成的平台。
以windows、arm、linux版本编译，如下：

```
$ GOOS=linux GOARCH=amd64 make release-client
$ GOOS=windows GOARCH=amd64 make release-client
$ GOOS=linux GOARCH=arm make release-client
```

不同平台使用不同的 GOOS 和 GOARCH，GOOS为go编译出来的操作系统 (windows,linux,darwin)，GOARCH, 对应的构架 (386,amd64,arm)

通过上面的步骤，将生成所有客户端文件，客户端文件放在对于的文件夹中，如windows 64位的为：windows_amd64，linux客户端在bin目录下的ngrok文件。

完成之后，把相应的客户端文件使用SFTP或其他方式分发到客户端电脑上面，比如我用的windows电脑，就把`windows_amd64/ngrok.exe`文件复制过去。

### 启动ngrokd服务器

请将 bin/ngrokd 放入PATH环境变量中，启动命令：
```
nohup ngrokd -domain=ngrok.xncoding.com -httpAddr=:2222 -httpsAddr=:3333 -tunnelAddr=":7777" &
```

`-domain`为你的服务域名，`-httpAddr`为http服务端口地址，访问形式为`xxx.ngrok.xncoding.com:2222`，也可设置为80默认端口，`-httpsAddr`为https服务，同上。

ngrokd还会开一个端口用来跟客户端通讯（可通过`-tunnelAddr=":xxx"` 指定），如果你配置了 iptables 规则，需要放行这个通讯端口(7777)上的 TCP 协议。

```
firewall-cmd --zone=public --add-port=7777/tcp --permanent
firewall-cmd --reload
```

### Nginx配置80端口转发

我们在微信开发时候不允许使用端口访问，那么最好使用nginx反向代理转发，首先申请一个`demo.ngrok.xncoding.com`的免费证书，然后修改nginx配置如下：
```
server {
    listen       80;
    listen       443 ssl http2;
    server_name  demo.ngrok.xncoding.com;

    charset utf-8;

    ssl_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/demo.ngrok.xncoding.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/chain.pem;

    access_log /var/log/nginx/ngrok.log main;
    error_log /var/log/nginx/ngrok_error.log error;

    location / {
        proxy_pass http://127.0.0.1:2222;
        proxy_redirect off;
        proxy_set_header Host       $http_host:2222;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        client_max_body_size 10m;
        client_body_buffer_size 128k;
        proxy_connect_timeout 90;
        proxy_read_timeout 90;
        proxy_buffer_size 4k;
        proxy_buffers 6 128k;
        proxy_busy_buffers_size 256k;
        proxy_temp_file_write_size 256k;
    }
}
```

但是！这里就有一个很烦躁的地方了，ngrokd 里面有一层自己的 Host 处理，于是 `proxy_set_header Host` 必须带上 ngrokd 所监听的端口，
否则就算请求被转发到对应端口上， ngrokd 也不会正确的处理。

带上端口号又会导致了另一个操蛋的问题：你请求的时候是`demo.ngrok.xncoding.com`， 
你在 web 应用中获取到的 Host 是 `demo.ngrok.xncoding.com:2222`，
如果你的程序里面有基于 Request Host 的重定向，就会被重定向到 `demo.ngrok.xncoding.com:2222` 下面去。

要完美的解决这个端口的问题，就需要让 ngrokd 直接监听 80 端口。

### 启用客户端

在刚刚复制过来的ngrok.exe客户端文件夹中，新建一个客户端配置`ngrok.cfg`：
```
server_addr: "ngrok.xncoding.com:7777"
trust_host_root_certs: false
```

本地启动一个SpringBoot的WEB工程，端口8092，然后通过下面命令启动客户端：
```
ngrok.exe -subdomain demo -config=ngrok.cfg -log=log.txt 8092
```

看到下面的画面说明连接成功了：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok03.png)

访问页面，浏览器中输入：`https://demo.ngrok.xncoding.com`，成功访问本地SpringBoot站点内容。

浏览器输入：`127.0.0.1:4040` 查看页面请求情况：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok02.png)


## 国内免费的ngrok

如果你自己没VPS，或者你机子上面80端口已经被nginx占用不想搞了，就直接使用免费的ngrok吧，
我推荐你使用<https://www.ngrok.cc/>。

比如我自己弄了个`yidao620.free.ngrok.cc`，启动本地客户端后，映射到本地的8092端口了，也还不错。

## 参考文章

* [从零教你搭建ngrok服务器](https://morongs.github.io/2016/12/28/dajian-ngrok/)
* [ngrok使用自己的证书通过https访问](https://www.jianshu.com/p/4b03fb532145)
* [搭建并配置优雅的 ngrok 服务实现内网穿透](https://yii.im/posts/pretty-self-hosted-ngrokd/)
* [使用Docker搭建Ngrok服务器实现内网穿透](https://blog.fengcl.com/2017/05/24/how-to-use-docker-build-ngrok-to-network-penetrate/)
* [搭建自己的Ngrok服务器, 并与Nginx并存](https://fengqi.me/unix/409.html)
