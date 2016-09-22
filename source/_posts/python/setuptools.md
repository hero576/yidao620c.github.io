---
title: "python核心 - 打包与发布"
date: 2015-10-26 22:22:22 +0800
comments: true
toc: true
categories: python
tags: [python]
---
当需要将写的程序打包分发出去的时候，就要使用到setuptools工具了，这里我通过一个简单例子来介绍它的使用方法

### 项目目录结构
这里我写了一个很简单的django程序来展示这种，目录结果如下
![](http://yidaospace.qiniudn.com/pysetup001.png)

项目最顶层的目录为“zspace”，其中与打包最相关的文件是setup.py，这里面最核心的文件就是这个setup.py了，我们看看里面写什么：<!--more-->
``` python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-

from setuptools import setup

setup(
    name='zspace',
    version='1.0.0',
    description='A auto test django project',
    url='https://github.com/yidao620c/core-python',
    author='Xiong Neng',
    author_email='yidao620@gmail.com',
    license='MIT',
    classifiers=[
        'Development Status :: 4 - Beta',
        'Intended Audience :: Developers',
        'Topic :: Software Development :: Build Tools',
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python :: 2.6',
        'Programming Language :: Python :: 2.7',
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.3',
        'Programming Language :: Python :: 3.4',
        'Programming Language :: Python :: 3.5',
    ],
    keywords='simple django project',
    packages=['autotest', 'zspace'],
)

```

* name为项目名称，和顶层目录名称一致;
* version是项目当前的版本，1.0.0.dev1表示1.0.0版，目前还处于开发阶段
* description是包的简单描述，这个包是做什么的
* url为项目访问地址，我的项目放在github上。
* author为项目开发人员名称
* author_email为项目开发人员联系邮件
* license为本项目遵循的授权许可
* classifiers有很多设置，具体内容可以参考官方文档
* keywords是本项目的关键词，理解为标签
* packages是本项目包含哪些包,我这里只有一个名词为hive的包

### 项目打包
cd zspace
python setup.py bdist_wheel

如果报错：invalid command 'bdist_wheel'，则先安装下wheel模块：
``` python
pip install wheel
```

执行完后，在顶层项目目录下将产生几个新的目录：
![](http://yidaospace.qiniudn.com/pysetup002.png)

### 注册PyPI帐号
如果没有账号需要先在PyPI网站上注册账号。
在您的本机用户下创建~/.pypirc文件，此文件中配置PyPI访问地址和账号。下面是我的.pypirc文件内容请根据自己的账号来修改。

```
[distutils]
index-servers = pypi

[pypi]
username:xiongneng
```

### 注册项目
```
python setup.py register
```

通过上面.pypirc文件中的配置，在PyPI上注册项目信息，成功注册之后，可以在PyPI上看到自己的项目名称：

### 上传项目
python setup.py sdist bdist_wheel upload

通过上面.pypirc文件中的配置，上传打包文件，可以在PyPI上看到上传的项目文件：

恭喜你成功将你的软件包上传至PyPI上面，全世界的人都可以通过pip来安装了：
```
pip install zspace
```

