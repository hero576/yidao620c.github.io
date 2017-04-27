---
layout: post
title: "Jenkins持续集成 - 蓝色海洋"
date: 2017-03-25 09:32:16 +0800
toc: true
categories: fullstack
tags: [jenkins]
---

Jenkins最新整了个`Blue Ocean`出来，我觉得有必要用单独一篇来介绍一下这个东西。

本篇会涉及到`Blue Ocean`的各个方面，包括他的`Dashboard`，查看分支和各个管道执行结果，
使用可视化编辑器来修改管道代码等。

`Blue Ocean`重新设计了用户使用Jenkins的方式，给我们带来极大的方便，同时也兼容自由风格的任务定义。<!--more-->

## 安装

可以在当前Jenkins环境下面安装`Blue Ocean`插件，具体步骤：

1. Login to your Jenkins server
2. Click Manage Jenkins in the sidebar then Manage Plugins
3. Choose the Available tab and use the search bar to find Blue Ocean
4. Click the checkbox in the Install column
5. Click either Install without restart or Download now and install after restart

## 启动

