---
title: EC2虚拟主机搭建SS
date: 2017-09-30 17:29:33
toc: true
categories: 朝花夕拾
tags: gitbook
---

作为一个离开 Google 生活就无法自理的人类，我曾经发帖、提问、翻遍各种网站，四处寻找靠谱的科学上网利器。
网上也买过好多，自己也用过一些开源免费的proxy，最后都会出现各种莫名其妙的问题，各种不稳定。

最后我选择自己动手，丰衣足食，利用EC2虚拟主机搭建SS，这个东东是由 [Clowwindy](https://github.com/Clowwindy) 开发的一款软件，
其作用本来是加密传输资料。当然，也正因为它加密传输资料的特性，使得XXX没法将由它传输的资料和其他普通资料区分开来，
也就不能干扰我们访问那些「不存在」的网站了。

如果你希望不花钱就能用上优质的服务──醒醒，别做梦了，免费和优质从来不可能划上等号。
不过想要共享或建立多账户来出售的话，能赚钱也说不定🙃 <!--more-->

## 创建EC2虚拟主机

先去亚马逊AWS上面注册一个账号：<https://amazonaws-china.com/cn/>

登录后进入EC2的控制台，选择左边的"实例" ——> 启动实例 ——> AWS Marketplace ——> 搜索"centos7"，
目前最新的版本是CentOS7.4

![](https://xnstatic-1253397658.file.myqcloud.com/ss01.png)

然后点击"continue"，默认选中符合条件的免费套餐。

![](https://xnstatic-1253397658.file.myqcloud.com/ss02.png)

然后点击"审核和启动"，这里编辑一下安全组信息，创建一个新安全组，类型里面选择"所有流量"即可。

启动之后，回到实例的界面，然后点击"连接"，复制上面的实例名

![](https://xnstatic-1253397658.file.myqcloud.com/ss03.png)

然后用xshell工具来连接，主机名选择上面实例详情的名称，使用密钥对来登录，用户名选择centos即可。

## 部署SS
好了，现在开始正式部署ss了，这里使用 [teddysun](https://teddysun.com/342.html) 的一键安装脚本。

先切换到root用户，可使用 `sudo passwd root` 先修改root密码，然后执行：

``` bash
wget --no-check-certificate https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
chmod +x shadowsocks.sh
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```

按照提示来，最后成功界面如下：

![](https://xnstatic-1253397658.file.myqcloud.com/ss05.png)

请把这些信息复制下来。

## TCP Fast Open

实际上只要具备上述四个信息，你就可以在自己的任意设备上进行登录使用了。但是为了更好的连接速度，你还需要多做几步。

首先是打开 TCP Fast Open，`vi /etc/rc.local` ，在最后增加如下内容：

```
echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```

然后修改`/etc/sysctl.conf`，在最后增加如下内容：
```
net.ipv4.tcp_fastopen = 3
```

再打开一个 Shadowsocks 配置文件，编辑`/etc/shadowsocks.json`，修改如下：
```
"fast_open":true
```

最后，输入以下命令重启 Shadowsocks：
```
/etc/init.d/shadowsocks restart
```

## 安装SS客户端

相比服务器端的安装，客户端的安装就简单了许多。首先，根据操作系统下载相应的客户端。

[Win 版客户端下载](https://github.com/shadowsocks/shadowsocks-windows/releases)

打开客户端，在「服务器设定」里新增服务器。然后依次填入服务器 IP、服务器端口、你设的密码和加密方式。

![](https://xnstatic-1253397658.file.myqcloud.com/ss06.png)

然后点击"启用系统代理"，选择PAC模式，在PAC中选择从xxx更新本地PAC，就可以实现科学上网了。

## 开启锐速

[锐速](https://github.com/91yun/serverspeeder) 是一个 TCP 加速软件，对 Shadowsocks 客户端和服务器端间的传输速度有显著提升。

「锐速」的一大优势是只需要在服务器端单边部署就行了，换句话说，你不需要再安装另外一个应用。

一键安装：

``` bash
wget -N --no-check-certificate https://github.com/91yun/serverspeeder/raw/master/serverspeeder.sh && bash serverspeeder.sh
```

安装上面官网的的安装步骤执行一键安装脚本会出现如下的错误信息：
```
前面的省略...
Complete!
=================================================
操作系统：CentOS 
发行版本：7.4 
内核版本：3.10.0-693.el7.x86_64 
位数：x64 
锐速版本：3.10.61.0 
=================================================
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 96179  100 96179    0     0  77305      0  0:00:01  0:00:01 --:--:-- 77314


锐速暂不支持该内核，程序退出.自动安装判断比较严格，你可以到http://www.91yun.org/serverspeeder91yun手动下载安装文件尝试不同版本

```

锐速只能适配几个特定的内核，我现在的CentOS7.4内核并不支持，那么就需要修改内核来适配了。具体步骤如下

**改核适配锐速**

### 监测VPS架构

``` bash
wget -N --no-check-certificate https://raw.githubusercontent.com/91yun/code/master/vm_check.sh && bash vm_check.sh
```

如果是kvm还是xen或者vmare则可以装锐速，如果是Openvz，则不可装锐速。

### Centos6安装内核（推荐使用）

CentOS 6支持安装锐速的内核：2.6.32–504.3.3.el6.x86_64

``` bash
uname -r #查看当前内核版本
rpm -ivh http://xz.wn789.com/CentOSkernel/kernel-firmware-2.6.32-504.3.3.el6.noarch.rpm
rpm -ivh http://xz.wn789.com/CentOSkernel/kernel-2.6.32-504.3.3.el6.x86_64.rpm --force
rpm -qa | grep kernel #查看是否安装成功
reboot #重启VPS
uname -r #当前使用内核版本
```

### Centos7安装内核（不太推荐）

CentOS 7支持安装锐速的内核：3.10.0-327.el7.x86_64

``` bash
yum install -y linux-firmware
rpm -ivh http://xz.wn789.com/CentOSkernel/kernel-3.10.0-229.1.2.el7.x86_64.rpm --force
rpm -qa | grep kernel #查看内核是否安装成功
reboot #重启VPS
uname -r #当前使用内核版本
```

锐速针对Centos 7的版本较少，推荐在Centos 6中安装。

成功界面如下，看到license信息过期时间为"2034-12-31"就没问题了。

![](https://xnstatic-1253397658.file.myqcloud.com/ss07.png)

### 部署锐速

依然使用一键安装脚本，输入以下命令：
``` bash
wget -N --no-check-certificate https://raw.githubusercontent.com/91yun/serverspeeder/master/serverspeeder-all.sh && bash serverspeeder-all.sh
```

安装需要一段时间，等待一会。

至此，整个搭建过程就大功告成了！接下来，尽情地享受起飞的速度吧😄

---------------

参考文章

* [科学上网的终极姿势](https://zoomyale.com/2016/vultr_and_ss/)
* [CentOS改核适配锐速](http://www.topchinaz.com/centos-%E6%94%B9%E6%A0%B8%E9%80%82%E9%85%8D%E9%94%90%E9%80%9F%EF%BC%88digitalocean-centos7-4%EF%BC%89/)

