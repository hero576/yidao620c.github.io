---
title: "Scrapy笔记（9）- 部署"
date: 2016-03-21 00:16:12 +0800
comments: true
toc: true
categories: python
tags: [scrapy]
---

## 两种方案
本篇主要介绍两种部署爬虫的方案。如果仅仅在开发调试的时候在本地部署跑起来是很容易的，不过要是生产环境，爬虫任务量大，并且持续时间长，那么还是建议使用专业的部署方法。主要是两种方案：

* [Scrapyd](http://doc.scrapy.org/en/1.0/topics/deploy.html#deploy-scrapyd) 开源方案
* [Scrapy Cloud](http://doc.scrapy.org/en/1.0/topics/deploy.html#deploy-scrapy-cloud) 云方案

## 部署到Scrapyd
[Scrapyd](http://doc.scrapy.org/en/1.0/topics/deploy.html#deploy-scrapyd)是一个开源软件，用来运行蜘蛛爬虫。它提供了HTTP API的服务器，还能运行和监控Scrapy的蜘蛛<!--more-->

要部署爬虫到Scrapyd，需要使用到[scrapyd-client](https://github.com/scrapy/scrapyd-client)部署工具集，下面我演示下部署的步骤

Scrapyd通常以守护进程daemon形式运行，监听spider的请求，然后为每个spider创建一个进程执行`scrapy crawl myspider`,同时Scrapyd还能以多进程方式启动，通过配置`max_proc`和`max_proc_per_cpu`选项

### 安装
使用pip安装
``` bash
pip install scrapyd
```
在ubuntu系统上面
``` bash
apt-get install scrapyd
```

### 配置
配置文件地址，优先级从低到高

* /etc/scrapyd/scrapyd.conf (Unix)
* /etc/scrapyd/conf.d/* (in alphabetical order, Unix)
* scrapyd.conf
* ~/.scrapyd.conf (users home directory)

具体参数参考[scrapyd配置](http://scrapyd.readthedocs.org/en/latest/config.html)

简单的例子
```
[scrapyd]
eggs_dir    = eggs
logs_dir    = logs
items_dir   =
jobs_to_keep = 5
dbs_dir     = dbs
max_proc    = 0
max_proc_per_cpu = 4
finished_to_keep = 100
poll_interval = 5
bind_address = 0.0.0.0
http_port   = 6800
debug       = off
runner      = scrapyd.runner
application = scrapyd.app.application
launcher    = scrapyd.launcher.Launcher
webroot     = scrapyd.website.Root

[services]
schedule.json     = scrapyd.webservice.Schedule
cancel.json       = scrapyd.webservice.Cancel
addversion.json   = scrapyd.webservice.AddVersion
listprojects.json = scrapyd.webservice.ListProjects
listversions.json = scrapyd.webservice.ListVersions
listspiders.json  = scrapyd.webservice.ListSpiders
delproject.json   = scrapyd.webservice.DeleteProject
delversion.json   = scrapyd.webservice.DeleteVersion
listjobs.json     = scrapyd.webservice.ListJobs
daemonstatus.json = scrapyd.webservice.DaemonStatus
```

### 部署
使用[scrapyd-client](https://github.com/scrapy/scrapyd-client)最方便



## 部署到Scrapy Cloud

