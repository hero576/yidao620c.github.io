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

搭建ngrok服务需要有一天外网服务器及一个域名解析到外网服务器上，我已经有了一个`xncoding.com`域名，并且拥有一台腾讯云主机。

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

通过上面的步骤，将生成所有客户端文件，客户端文件放在对于的文件夹中，
如windows 64位的为：windows_amd64，linux客户端在bin目录下的ngrok文件。

完成之后，把相应的客户端文件使用SFTP或其他方式分发到客户端电脑上面，
比如我用的windows电脑，就把`windows_amd64/ngrok.exe`文件复制过去。

### 启动ngrokd服务器

请将 bin/ngrokd 放入环境变量中，启动命令：
```
nohup ngrokd -domain=ngrok.xncoding.com -httpAddr=:5442 -httpsAddr=:5443 &
```

其中，`-domain`为你的ngrok服务域名，`-httpAddr`为http服务端口地址，访问形式为：`xxx.ngrok.xncoding.com:5442`，
也可设置为80默认端口，-httpsAddr为https服务，同上。

ngrokd还会开一个4443端口用来跟客户端通讯（可通过`-tunnelAddr=":xxx"` 指定），如果你配置了 iptables 规则，需要放行这几个端口上的 TCP 协议。

### Nginx配置80端口转发

我们在微信开发时候不允许使用端口访问，那么最好使用nginx反向代理转发，配置如下：
```
server {
    listen 80;
    server_name *.ngrok.xncoding.com;
    charset utf-8;

    access_log /var/log/nginx/ngrok.log main;
    error_log /var/log/nginx/ngrok_error.log error;

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host:5442;
        proxy_redirect off;
        proxy_pass http://127.0.0.1:5442;
    }
}

server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  *.ngrok.xncoding.com;

    access_log /var/log/nginx/ngrok.log main;
    error_log /var/log/nginx/ngrok_error.log error;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host:5443;
        proxy_redirect off;
        proxy_pass https://127.0.0.1:5443;
    }
}
```

但是！这里就有一个很烦躁的地方了，ngrokd 里面有一层自己的 Host 处理，于是 `proxy_set_header Host`
必须带上 ngrokd 所监听的端口，否则就算请求被转发到对应端口上， ngrokd 也不会正确的处理。

带上端口号又会导致了另一个操蛋的问题：你请求的时候是`ngrok.xncoding.com`，
你在 web 应用中获取到的 Host 是 `ngrok.xncoding.com:8081`，如果你的程序里面有基于 Request Host 的重定向，
就会被重定向到 `ngrok.xncoding.com:8081` 下面去。

要完美的解决这个端口的问题，就需要让 ngrokd 直接监听 80 端口。

还有一种方案是，如果你的VPS支持双网卡就好办了，nginx监听外网80端口，ngrokd 监听内网80，让 nginx 将对应的请求转发到内网 80 上来。

比如：

* 内网 ip: 10.160.xx.xx
* 外网 ip: 112.124.xx.xx

启动 ngrokd：
```
ngrokd -tlsKey=server.key -tlsCrt=server.crt -domain=ngrok.xncoding.com -httpAddr=10.160.xx.xx:80 -httpsAddr=
```

配置 nginx：
```
# ngrokd.conf
server {
    listen      112.124.xx.xx:80;
    server_name *.ngrok.xncoding.com;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect  off;
        proxy_pass      http://10.160.xx.xx:80;
    }
}
```

由于我的腾讯云主机只是单网卡的，所以没办法实现。这时候推荐使用后面我讲的Docker方案了。

另外在http配置处还需要加入一行：
```
resolver 127.0.0.1;
```

重新加载「nginx」
```
nginx -s reload
```

### 启用客户端

在刚刚复制过来的ngrok.exe客户端文件夹中，新建一个客户端配置：
```
server_addr: "ngrok.xncoding.com:4443"
trust_host_root_certs: false
```

本地启动一个SpringBoot的WEB工程，端口8092，然后通过下面命令启动客户端：
```
ngrok.exe -subdomain demo -config=ngrok.cfg -log=log.txt 8092
```

看到下面的画面说明连接成功了：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok03.png)

访问页面，浏览器中输入：`demo.ngrok.xncoding.com`，成功访问本地SpringBoot站点内容。

浏览器输入：`127.0.0.1:4040` 查看页面请求情况：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok02.png)

## HTTPS证书

如果直接使用https访问的话，浏览器会出现警告，这里说明一下如何让网站支持https而浏览器不拒绝。

1、首先去买一个ssl证书，或者申请一个免费的，比如我最喜欢的Lets Encrypt证书。然后把你的证书上传到ngork服务端所在的服务器。
（我的证书是一个crt和一个key文件）

2、将你的域名泛解析到你的服务器。

3、使用下面的命令运行服务端：
```
nohup ngrokd -domain=ngrok.xncoding.com -tlsKey="/path/ngrok/your.key" -tlsCrt="/path/ngrok/your.crt" -httpAddr=:5442 -httpsAddr=:5443 &
```

其实就是在你原先的命令上加tlskey和tlscrt的路径,这两个就是你的证书所在路径.

4、客户端cfg文件里，使用hostname+https的方式启动客户端（hostname就是你证书的域名）

然后在客户端第二行设置如下参数：
```
trust_host_root_certs: true
```

5、确认服务端的启动参数-domain以及客户端cfg文件中的server_addr和证书的域名是同一个，否则会报错误证书的错误。

6、如果你申请的是免费的证书，可能crt文件不带中间商和根证书，这时需要你去网站上把所有证书合在一起，
否则在linux上使用客户端会出现`certificate signed by unknown authority`的错误, 参考<http://m.ithao123.cn/content-2350159.html>

到此如果没有什么问题，你的网站就可以用https访问了，而且浏览器也不会再提示是不安全的网站了。

至此，ngrok内网穿透成功实现，幸福的汗水。

## 使用Docker

上面我讲到自己手动搭建的时候出现的端口映射问题，没办法解决。而最完美的方式是使用Docker + Nginx的方式。

### 安装docker

参考我的这篇[Docker入门](https://www.xncoding.com/2017/04/01/docker/docker01.html)来安装docker。

### 构建镜象

这里使用的是[hteen/docker-ngrok](https://github.com/hteen/docker-ngrok)

```
git clone https://github.com/hteen/docker-ngrok.git
cd docker-ngrok
docker build -t hteen/ngrok .
```

这里需要等待一段时间下载

### 运行镜象

先申请`ngrok.xncoding.com`这个域名的CA证书，免费的lets encrypted证书，然后修改脚本`server.sh`

```
-tlsKey=/etc/letsencrypt/live/ngrok.xncoding.com/privkey.pem -tlsCrt=/etc/letsencrypt/live/ngrok.xncoding.com/fullchain.pem
```
也就是将之前的证书变量改成你实际的证书路径即可。

然后运行：

```
docker run -idt --name ngrok-server \
-p 5442:80 -p 5443:443 -p 4443:4443 \
-v /data/ngrok:/myfiles \
-e DOMAIN='ngrok.xncoding.com' -e HTTP_ADDR=':5442' -e HTTPS_ADDR=':5443' hteen/ngrok /bin/sh /server.sh
```

这里会把主机的5442端口映射到Docker容器中的80端口，讲5443端口映射到443端口，同时将本机的/data/ngrok文件夹映射到docker容器的/myfiles目录。

运行后，会要等一段时间，因为要编译客户端。

### Docker容器的https

关于 https 的支持

由于 ngrok 工作是通过分配 subdomain 的方式，所以我们实际使用到的域名都是 `ngrok.xncoding.com` 的子域名，
如 `demo.ngrok.xncoding.com` 如果要对这个子域名启用 https 服务，那么至少需要三点支持：

1. ngrok 支持 https， 这个默认就是开启的
1. `demo.ngrok.xncoding.com` 也需要有证书或包含在一个泛域名证书中
1. 浏览器（或其他终端）信任 `demo.ngrok.xncoding.com` 的根证书

根据这三点要求，我们重新解读上面三种证书的处理方式：

第一种：由于免费证书是单域名证书，所以你需要给可能会用到二级域名也签上证书才行，当然，如果够钱，买个包含所有二级域名的证书也是可以的

第二种：自签名证书很容易做到第二点，然而并无卵用，除了自编译的 ngrok-client外谁也不认这个证书

第三种：可以支持 https，但是要所有用户（包括访问用户）都添加根证书这种要求，略微有点…

综上所述，我们选择放弃了 https ，因为日常使用并没有强制要求 https 的情况，能跑就够了，要什么自行车。

### 配置Nginx

启动之后需要在nginx.conf 添加两条反向代理配置：
```
server {
    listen       80;
    server_name  ngrok.xncoding.com *.ngrok.xncoding.com;
    location / {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://10.122.22.2:5442;
    }
}
server {
    listen       443;
    server_name  *.ngrok.xncoding.com;
    ssl_certificate /etc/letsencrypt/live/demo.ngrok.xncoding.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/demo.ngrok.xncoding.com/privkey.pem;
    ssl_dhparam /etc/ssl/private/dhparam.pem;
    location / {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://10.122.22.2:5443;
    }
}
~ 
```

其中`10.122.22.2`是我的内网IP地址。

### 运行客户端

稍后会在`/data/ngrok/bin/`目录下生成客户端程序，每个平台的版本都有。以windows64位来说，
在windows_amd64目录下，拷贝到自己的windows电脑上。

新建配置文件ngrok.cfg，跟ngrok.exe同级目录，里面的内容跟之前讲的一样：
```
server_addr: "ngrok.xncoding.com:4443"
trust_host_root_certs: false
```

然后打开windows的命令行，cd到ngrok.exe所在的目录中，到这个运行：
```
ngrok -config=ngrok.cfg -subdomain=demo -log=log.txt 8092
```

看到下面的结果表示成功了：

![](https://xnstatic-1253397658.file.myqcloud.com/ngrok04.png)

然后再打开`http://demo.ngrok.xncoding.com`看看，发现不会像之前那样出现端口了。


### 注意事项

因为ngrok有心跳机制，每次心跳均会产生日志，所以以docker方式运行，会产生很多日志。
实测试中，大概每个星期会产生100M的日志文件。

查年docker日志文件位置`sudo docker inspect <id> | grep LogPath`

查看大小`sudo ls -lh /var/lib/docker/containers/<id>/<id>-json.log`

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
