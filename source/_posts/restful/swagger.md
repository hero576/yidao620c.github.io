---
title: "使用Swagger生成RESTful API文档"
date: 2017-06-09 12:32:08 +0800
comments: true
toc: true
categories: restful
tags: [swagger]
---
REST API都是要对外提供服务的，那么文档是必须的。经常要给其他人员提供文档，每次都是要不断的维护word/excel的文件，挺麻烦的。
能不能做到自动生成呢？答案是可以的，swagger就是这样的一个组件帮助我们快速生成，
让开发人员只需要关注功能的开发即可，后续的工作就交给Swagger就好了。

Swagger是一个简单但功能强大的API表达工具。它具有地球上最大的API工具生态系统，数以千计的开发人员，
使用几乎所有的现代编程语言，都在支持和使用Swagger。使用Swagger生成API，我们可以得到交互式文档，
自动生成代码的SDK以及API的发现特性等。

2.0版本已经发布，Swagger变得更加强大。值得感激的是，Swagger的源码100%开源在[github](https://github.com/swagger-api)。

我演示的是为Jersey2自动生成API文档，更多的请参考[官方文档](http://swagger.io/docs/)。<!--more-->



