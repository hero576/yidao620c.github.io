---
title: 使用hexo搭建github博客
date: 2016-03-06 19:47:53
toc: true
categories: 朝花夕拾
tags: hexo
---

最今天我又折腾了我的博客，将它从octopress迁移到hexo上来。之前还专门写了一篇怎样利用octopress搭建博客的文章，
最近试用了一下hexo，毫不犹豫的迁移过来了，实在是忍受不了octopress的速度，还有稳定性，经常莫名其妙的出错。

hexo是一个台湾人做的基于Node.js的静态博客程序，优势是生成静态文件的速度非常快，支持markdown，
我最终选定它的原因是它速度快而且不容易出错，并且可以一键部署到github或者其它静态服务器上去。折腾了一天总算搞定。<!--more-->

## 安装

我这个教程hexo3版本，本地使用Windows7系统，在IDEA上面写Markdown的博客，爽歪歪。

### 安装依赖软件
[Node.js](https://nodejs.org/en): node.js用来创建hexo博客框架的

如果是windows系统，可以直接下载安装文件安装。如果是linux系统，可以使用下面的命令行：

```
wget https://nodejs.org/dist/v6.10.0/node-v6.10.0.tar.gz
tar xzvf node-v* && cd node-v*
yum install gcc gcc-c++
./configure --prefix=/usr/local/nodejs
make && make install  #这步时间很久20分钟
node --version  # 如果命令找不到就将/usr/local/nodejs/bin加入PATH中
```

最新的node.js已经集成了npm，不过需要经常更新：
```
npm install npm@latest -g
npm --version
```

## 换淘宝源
```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```
之后使用cnpm命令代替npm命令，或者你直接通过添加 npm 参数 alias 一个新命令:
```
alias cnpm="npm --registry=https://registry.npm.taobao.org \
--cache=$HOME/.npm/.cache/cnpm \
--disturl=https://npm.taobao.org/dist \
--userconfig=$HOME/.cnpmrc"
```

[Git客户端](http://git-scm.com/download/win): 把本地的hexo内容提交到github上去

### 安装hexo
利用 npm 命令即可安装。打开窗口控制台，输入安装hexo命令：
``` bash
cnpm install -g hexo
```

### 初始化
安装完成后，在你喜爱的文件夹下（如D:\hexo），控制台执行以下指令，hexo会自动在目标文件夹建立网站所需要的所有文件。
``` bash
hexo init
```

安装依赖包：
``` bash
cnpm install
```

一切准备就绪让我们试验下，在D:\hexo内执行以下命令：
``` bash
hexo g
hexo s
```
然后用浏览器访问http://localhost:4000， 此时，你应该看到了一个漂亮的博客了，当然这个博客只是在本地的，别人是看不到的，hexo3.0使用的默认主题是landscape。


## 发布到github上面

首先你得有github账号，如果没有就去注册个，很简单的步骤。

### 创建repository

repository相当于一个仓库，用来放置你的代码文件。登陆进入Github，并进入个人页面，选择repositories，然后New一个repository。
创建时，只需要填写Repository name即可。格式必须为yourname.github.io，比如我的是yidao620c.github.io

### 将本地博客部署上去

先修改D:\hexo下的_config.yml文件,记得一点，hexo的配置文件中任何’:’后面都是带一个空格的

```
deploy:
    type: git
    repository: https://github.com/yidao620c/yidao620c.github.io.git
    branch: master
```

我刚开始是部署到github上面，现在我部署到自己的腾讯云主机上面去了，
原理都一样，在腾讯云主机上面创建一个git服务即可。然后上面的`repository`改成自己的git服务器地址。

如果你是第一次使用Github或者是已经使用过，但没有配置过SSH，则可能需要配置一下SSH。
在Git Bash输入以下指令（任意位置点击鼠标右键），检查是否已经存在了SSH keys。
``` bash
ls -al ~/.ssh
```

如果不存在就没有关系，如果存在的话，直接删除.ssh文件夹里面所有文件

输入以下指令（邮箱就是你注册Github时候的邮箱）后回车，出现提示让你输入的时候一直按回车：
``` bash
ssh-keygen -t rsa -C "yidao620@gmail.com"
```

然后键入以下指令：
``` bash
eval `ssh-agent -s`
ssh-add
```
到了这一步，就可以添加SSH key到你的Github账户了。键入以下指令，拷贝Key（先拷贝了，等一下可以直接粘贴，不放心的在执行下面命令后，先黏贴在记事本上）：
``` bash
clip < ~/.ssh/id_rsa.pub
```
然后到Github里面，点击右上角的设置图标Settings,找到SSH keys,Ttile随便你命名，Key就黏贴上你刚才复制的key,然后点Add SSH key，最后会让你重新输入下gitHub的密码
最后还是测试一下吧，键入以下命令：
``` bash
ssh -T git@github.com
```
你可能会看到有警告，输入“yes”就好

还要安装hexo-deployer-git这个模块:
``` bash
npm install hexo-deployer-git --save
```

以上就表示SSH配置好了，执行以下命令部署到Github上。
``` bash
hexo g
hexo d
```
提示输入gitHub的账号密码，就能访问你得博客网站了。我的是: yidao620c.github.io

## 发表一篇文章
在你的D:\hexo目录下面，控制台执行命令：
``` bash
hexo new "Hello World"
```

会自动在D:\hexo\source\_posts文件夹生成一个hello-world.md文件，用编辑器打开，在里面写文章。
记住hexo中写文章使用的是Markdown语法，自己去google下这个语法吧，很方便很强大。里面的初始内容
```
title: Hello World #可以改成中文的，如“新文章”
date: 2016-03-06 16:04:09 #发表日期，一般不改动
categories: blog #文章文类
tags: [文章, 测试] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
#这里是正文，用markdown写，你可以选择写一段显示在首页的简介后，加上
<!--more-->，在<!--more-->之前的内容会显示在首页，之后的内容会被隐藏，当游客点击Read more才能看到。
```

写完文章后，你可以使用`hexo g`生成静态文件。`hexo s`在本地预览效果。`hexo d`同步到github

## hexo主题及其配置

如果你不喜欢默认主题，那么可以去hexo官网找更多的[主题](https://hexo.io/themes/)。
我这里选择NexT主题说明一下，这个是一个非常简洁的主题，我喜欢简单的东西。

Next主题官网：<http://theme-next.iissnan.com/>

### 安装主题

``` bash
$ cd your-hexo-site
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```

### 启用主题
修改你的博客根目录下的config.yml配置文件中的theme属性，将其设置为next

### 更新主题
``` bash
cd themes/next
git pull
```
更新好后，本地启动起来效果
``` bash
hexo server -g  #生成加预览
```

### 主题的_config.yml配置
具体配置请直接参考[开始使用NexT主题](http://theme-next.iissnan.com/getting-started.html)。
同时我对这个主题进行了很多的修改让它看上去更加符合自己的审美观，如果对我博客的主题感兴趣可以直接在我的github页面拉取即可。

## 启用disqus评论

最好的评论系统disqus因为你懂的原因国内不能访问，找了好久，最后通过[disqus-proxy](https://github.com/ciqulover/disqus-proxy)完美实现。

按照readme文档来配置，几个重要步骤说明一下。

1、首先去disqus添加一个网站，记住你申请的网站shortname

在主题配置里面开启评论
``` yml
disqus_shortname: xncoding
```

然后在全局配置里面开启：

``` yml
disqus_proxy:
  shortname: xncoding
  username: xiongneng
  host: yourserver.com
  port: 443
```

注意上面我设置的端口是443，后面我还要配置https访问。

2、disqus上面申请application，获取`Secret Key`，这个要记住，后面服务器配置用到。

3、后端配置

我再EC2服务器上面申请了一个虚拟机，后端使用Node.js，需要Node.js版本7.6以上

``` bash
su root
curl -sL https://rpm.nodesource.com/setup_7.x | bash -
yum install nodejs

git clone https://github.com/ciqulover/disqus-proxy

npm i --production
```

4、需要启动https访问，用nginx来反向代理`disqus proxy`，
参考[CentOS7配置自签名的SSL](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-on-centos-7)

重要步骤：
``` bash
sudo yum install nginx
systemctl start nginx
systemctl enable nginx

# If you have a firewalld firewall running
sudo firewall-cmd --add-service=http
sudo firewall-cmd --add-service=https
sudo firewall-cmd --runtime-to-permanent

# If have an iptables firewall running
sudo iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT

mkdir /etc/ssl/private
chmod 700 /etc/ssl/private
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt

# 上面很多提示最重要的一步是：Common Name (e.g. server FQDN or YOUR name) []:server_IP_address，配置成你的域名
openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

最后的添加的ssl配置如下：
```
server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name ec2-xxxxx.compute.amazonaws.com;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    location / {
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_pass http://127.0.0.1:5509$request_uri;
    }
}

```

然后测试并重启nginx：
```
nginx -t
systemctl restart nginx
```

对于那些连不上disqus的用户会显示一个精简版的评论框。

### 切换至畅言
使用disqus后发现还是有不少问题，首先是https因为不少认证过的，在浏览器里面有警告。
另外匿名评论我在管理后台看不到，不知道什么原因。最后还是切换至国内的畅言评论。

参考这篇 <http://www.jianshu.com/p/5888bd91d070>

先申请畅言，并且通过审核。获取到畅言评论的`APP ID` 和`APP KEY`，复制下来。

主题配置文件`_config.yml`中添加一个简单的配置即可：

```
# changyan
changyan:
  enable: true
  appid: your_appid
  appkey: your_appkey
```

## 多台电脑同时维护博客

之前利用了 Hexo + Github搭建了自己的博客网站，但问题来了：如何在多台电脑上对此博客进行维护？

### 思路

* 对于一个已创建的博客目录，其已通过hexo建立了public目录下的内容与远程仓库yidao620c@github.com的master分支的连接关系，通过hexo的hexo deploy命令即可保持更新；
* 因此只需要在此仓库yidao620c@github.com上再创建一个分支如source，将其与hexo下其他文件如node_modules、themes、scaffolds等（即除了public目录及.gitignore包含的文件外）进行同步绑定。
* 在不同的电脑下设置好hexo环境，通过hexo命令维护master分支，通过git命令维护source分支即可。

### 具体步骤

（假定最初创建博客为A，其他另一个为B)

1) 在A中的git_blog目录下，建立source分支：
``` bash
$ git branch source // 创建source分支
$ git checkout source // 切换到source分支
$ git remote -v //查看远程分支名字
 // 我的内容为如下，说明已绑定
 // origin  git@github.com:yidao620c/yidao620c.github.com.git (fetch)
 // origin	git@github.com:yidao620c/yidao620c.github.com.git (push)
修改.gitignore文件，添加"public/"字段至其中。
$ git push origin source // 将当前git_blog下的内容push到Github上的远程仓库的source分支（会自动创建）上
```

2) 在B中建立git_blog目录，安装npm，安装Hexo，添加SSH，然后在B中建立本地的source分支，并与远程的source分支进行了绑定。
git_blog下的yidao620c.github.com文件夹里保存了和远程source分支相同的内容。
``` bash
$ git clone -b source git@github.com:yidao620c/yidao620c.github.com.git
$ cd yidao620c.github.com
$ git branch -a
$ git checkout -b source origin/source
$ cnpm install hexo
$ cnpm install
$ cnpm install hexo-deployer-git --save
$ cnpm install hexo-renderer-jade --save
$ cnpm install hexo-renderer-sass --save
```

**重要记录**

千万别执行`hexo init`这个命令啊，同时`Next`主题的安装步骤还是需要的。

另外hexo3.4.3版本好像生成toc的时候有点问题，我退回到3.3.9就正常。

首先`package.json`改成绝对版本号：
``` json
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": "3.3.9"
  },
  "dependencies": {
    "hexo": "3.3.9",
    "hexo-algolia": "^1.2.3",
    "hexo-deployer-git": "^0.3.1",
    "hexo-generator-archive": "^0.1.5",
    "hexo-generator-category": "^0.1.3",
    "hexo-generator-feed": "^1.2.2",
    "hexo-generator-index": "^0.2.0",
    "hexo-generator-searchdb": "^1.0.8",
    "hexo-generator-tag": "^0.2.0",
    "hexo-renderer-ejs": "^0.3.0",
    "hexo-renderer-jade": "^0.4.1",
    "hexo-renderer-marked": "^0.3.0",
    "hexo-renderer-sass": "^0.3.2",
    "hexo-renderer-stylus": "^0.3.1",
    "hexo-server": "^0.2.0"
  }
}
```

然后把hexo卸载了，重新安装指定版本，在另外一个目录执行：
```
cnpm uninstall hexo -g
cnpm install hexo@3.3.9 -g
```

然后再回到hexo目录执行：
```
cnpm install
```

### 使用方法

如果是新电脑，先把source分支clone下来：
```bash
git clone -b source --single-branch https://github.com/yidao620c/yidao620c.github.io.git
```

在任意一台mac操作，都需先切换并保持在source分支上。使用git命令管理source文件；使用hexo命令进行同步至远程master分支，无需处理本地master分支

1. 在A中使用Hexo的new、g、d方法添加、生成、部署新的博客，内容都会被同步自动放到Github的master分支上
2. 在A中使用git命令的`push origin source`同步source到远程source分支
3. 同样保证B的git当前在git_blog下的source分支下

先使用：
``` bash
git pull origin source
```

获得Github的source分支上的最新版本，再使用：
``` bash
$ git add -u
$ git commit -m ""
$ git push origin source
```

将新内容提交至Github的source分支上，完成source管理。

### 其他说明
``` bash
git add -f xxx // 强制添加文件/文件夹为tracked状态
git rm --cache xxx // 解除文件为tracked状态
git rm -r --cache xxx // 解除文件夹为tracked状态
```

## 添加网站的RSS订阅

### 安装hexo-generator-feed

``` bash
cnpm install hexo-generator-feed --save
```
安装完后，会在node_modules目录下生成hexo-generator-feed目录

### 配置根目录的_config.yml

```
# Extensions
## Plugins: http://hexo.io/plugins/
#RSS订阅
plugin:
- hexo-generator-feed
#Feed Atom
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
```
其中，feed是可选项，可配可不配！然后根据各个主题的说明添加一个RSS订阅链接即可

## 自定义分页
安装三个插件：归档，标签，分类。然后将每个分页显示文章数都变大点
``` bash
npm install hexo-generator-archive --save
npm install hexo-generator-tag --save
npm install hexo-generator-category --save
```
三者各自的配置：
```
archive_generator:
  per_page: 100
  yearly: true
  monthly: true
  daily: false
tag_generator:
  per_page: 100
category_generator:
  per_page: 100
```

## 绑定自己的域名
Github Page绑定自己域名，在source/目录下面新建`CNAME`文件，里面写入你自己的域名比如
```
www.pycoding.com
```
去自己的域名运营商处添加CNAME类型的DNS记录:
```
CNAME: @        =>  yidao620c.github.io
CNAME: www      =>  yidao620c.github.io
```
如果是DNSPod，那么后面多加个点
```
@，CNAME，cmback.github.io.
www，CNAME，cmback.github.io.
```

## 站内搜索
最简单是直接开启`Local Search`

安装 `hexo-generator-searchdb`，在站点的根目录下执行以下命令：
```
cnpm install hexo-generator-searchdb --save
```

全局配置文件_config.yml中定义搜索页面：
``` html
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

在NexT的_config.yml配置中开启这个本地搜索：
```
# Local search
local_search:
  enable: true
```

## algolia搜索
使用一段时间的`Local Search`后发现还是不太好用，后来比较了一下Swiftype、 微搜索、Local Search 和 Algolia，
发现`algolia`比较好。安装`NexT`文档配置步骤如下：

第一步，先注册Algolia，创建Index

前往 [Algolia 注册页面](https://www.algolia.com/)，注册一个新账户。可以使用 GitHub 或者 Google 账户直接登录，
注册后的 14 天内拥有所有功能（包括收费类别的）。之后若未续费会自动降级为免费账户，免费账户总共有 10,000 条记录，
每月有 100,000 的可以操作数。注册完成后，创建一个新的 Index，这个 Index 将在后面使用。

第二步，安装Algolia
```
npm install --save hexo-algolia
```

第三步，获取Key，更新站点配置

在 Algolia 服务站点上找到需要使用的一些配置的值，包括 `ApplicationID`、`Search-Only API Key`、`min API Key`。
注意，Admin API Key 需要保密保存。点击ALL API KEYS 找到新建INDEX对应的key， 编辑权限，
在弹出框中找到ACL选择勾选`Search`、`Add records`、`Delete records`、`List indices`、`Delete index`权限，点击update更新。

![](https://xnstatic-1253397658.file.myqcloud.com/hexo20.png)

编辑站点配置文件，新增以下配置：
```
algolia:
  applicationID: 'IQUD4IM8MM'
  indexName: 'blogindex'
  chunkSize: 5000
```

替换除了 chunkSize 以外的其他字段为在 Algolia 获取到的值。

第四步，更新Index

当配置完成，新增一个环境变量，windwos和linux配置环境变量我就不讲了。

```
HEXO_ALGOLIA_INDEXING_KEY=Search-Only API key
```

在站点根目录下执行
```
$ hexo algolia
```

来更新 Index，请注意观察命令的输出。
```
$ hexo algolia
INFO  [Algolia] Testing HEXO_ALGOLIA_INDEXING_KEY permissions.
INFO  Start processing
INFO  [Algolia] Identified 152 pages and posts to index.
INFO  [Algolia] Indexing chunk 1 of 4 (50 items each)
INFO  [Algolia] Indexing chunk 2 of 4 (50 items each)
INFO  [Algolia] Indexing chunk 3 of 4 (50 items each)
INFO  [Algolia] Indexing chunk 4 of 4 (50 items each)
INFO  [Algolia] Indexing done.
```

注意，每次更新文章后记得执行`hexo algolia`更新索引哦。

第五步，主题集成

更改主题配置文件，找到 Algolia Search 配置部分：
```
# Algolia Search
algolia_search:
  enable: false
  hits:
    per_page: 10
  labels:
    input_placeholder: Search for Posts
    hits_empty: "We didn't find any results for the search: ${query}"
    hits_stats: "${hits} results found in ${time} ms"
```

将 `enable` 改为 `true` 即可，根据需要你可以调整 `labels` 中的文本。


## 阅读次数统计

使用LeanCloud为文章添加阅读次数统计，请参考文章[为NexT主题添加文章阅读量统计功能](https://notes.wanghao.work/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html#%E9%85%8D%E7%BD%AELeanCloud)

## FAQ

* 遇到有大括号的代码块，如果多行的不用管，如果单行的就单个反引号，并且在里面加raw标签，比如{% raw %} `{{test}}` {% endraw %}
* 关闭hexo的将回车当换行做法是用正常的markdown两个回车当换行，在全局_config.yml中添加配置
```
  marked:
    gfm: true
    breaks: false
```

## 参考

* [hexo干货系列](http://tengj.top/tags/hexo/)
* [大道至简——Hexo简洁主题推荐](https://www.haomwei.com/technology/maupassant-hexo.html)

