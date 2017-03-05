---
title: "使用ReadtheDocs托管文档"
date: 2017-01-22 20:16:09 +0800
comments: true
toc: true
categories: fullstack
tags: [sphinx, readthedocs]
---
[Read the Docs](https://readthedocs.org/)是一个在线文档托管服务，
你可以从各种版本控制系统中导入文档，如果你使用[webhooks](http://docs.readthedocs.io/en/latest/webhooks.html)，
那么每次提交代码后可以自动构建并上传至readthedocs网站，非常方便。

一般来讲，这个非常适合写软件文档以及编写一些教程、电子书之类。对于一些一两篇文章就能写清楚的可以记笔记或写博客，
但是如果要写成一个系列的，不如写成一本书的形式，更美观，也更系统。<!--more-->

现有的写电子书的方式，有以下几个解决方案，优劣势也很明显：

* 写博客，跟散文堆在一起，不便索引。
* GitHub Wiki，适合做知识整理，但排版一般，不方便查看。
* GitBook，样式不好看，访问速度慢。

经过比较最后锁定Sphinx + GitHub + ReadtheDocs 作为文档写作工具，
用 Sphinx 生成文档，GitHub 托管文档，再导入到 ReadtheDocs。

## Sphinx
Sphinx是一个基于Python的文档生成项目，最早只是用来生成 Python 官方文档，随着工具的完善，
越来越多的知名的项目也用他来生成文档，甚至完全可以用他来写书采用了reStructuredText作为文档写作语言,
不过也可以通过模块支持其他格式，待会我会介绍怎样支持MarkDown格式。

### 安装Sphinx:
``` bash
pip install sphinx sphinx-autobuild sphinx_rtd_theme
```
这一步时间会安装很多python依赖，耐心等等..

### 初始化:
``` bash
# 创建文档根目录
mkdir -p /root/work/scrapy-cookbook
cd scrapy-cookbook/
# 可以回车按默认配置来写
sphinx-quickstart
```
下面是我填写的，其他基本上默认即可：

> Separate source and build directories (y/n) [n]:y
> Project name: scrapy-cookbook
> Author name(s): Xiong Neng
> Project version []: 0.2
> Project release [1.0]: 0.2.2
> Project language [en]: zh_CN

安装软件tree查看目录树结构：
``` bash
yum install tree
```

然后运行 `tree -C .` 查看生成的sphinx结构:
```
.
├── build
├── make.bat
├── Makefile
└── source
    ├── conf.py
    ├── index.rst
    ├── _static
    └── _templates
```

添加一篇文章，在source目录下新建hello.rst，内容如下:
```
hello,world
=============
```

`index.rst` 修改如下:
```
Contents:
.. toctree::
   :maxdepth: 2

   hello
```

### 更改主题 sphinx_rtd_theme
更改source/conf.py:
``` python
import sphinx_rtd_theme
html_theme = "sphinx_rtd_theme"
html_theme_path = [sphinx_rtd_theme.get_html_theme_path()]
```

### 预览效果
然后在更目录执行`make html`，进入`build/html`目录后用浏览器打开`index.html`
![](http://xnstatic-1253397658.cossh.myqcloud.com/rtd01.png)

toctree 支持多级目录,比如要想将python.rst,java.rst笔记在不同的目录,toctree这样设置:
```
Contents:

.. toctree::

   python/python
   swift/swift

```
注意中间的空行

## 支持markdown编写
通过[recommonmark](https://recommonmark.readthedocs.io/en/latest/) 来支持markdown
``` bash
pip install recommonmark
```

然后更改conf.py:
``` python
from recommonmark.parser import CommonMarkParser
source_parsers = {
    '.md': CommonMarkParser,
}
source_suffix = ['.rst', '.md']
```

### AutoStructify
如果想使用高级功能，可以添加AutoStructify配置，在`conf.py`中添加:
``` python
# At top on conf.py (with other import statements)
import recommonmark
from recommonmark.transform import AutoStructify

# At the bottom of conf.py
def setup(app):
    app.add_config_value('recommonmark_config', {
            'url_resolver': lambda url: github_doc_root + url,
            'auto_toc_tree_section': 'Contents',
            }, True)
    app.add_transform(AutoStructify)
```

网上有个详细配置: <https://github.com/rtfd/recommonmark/blob/master/docs/conf.py>

然后修改刚刚的`hello.rst`，改用熟悉的`hello.md`编写:
``` md

## hello world

### test markdown

```
再次运行`make html`后看效果，跟前面一样。

## GitHub托管
一般的做法是将文档托管到版本控制系统比如github上面，push源码后自动构建发布到readthedoc上面，
这样既有版本控制好处，又能自动发布到readthedoc，实在是太方便了。

先在GitHub创建一个仓库名字叫scrapy-cookbook，
然后在本地.gitignore文件中添加`build/`目录，初始化git，commit后，添加远程仓库。

具体几个步骤非常简单，参考官方文档：<https://github.com/rtfd/readthedocs.org>:

1. 在Read the Docs上面注册一个账号
2. 登陆后点击 "Import".
3. 给该文档项目填写一个名字比如 "scrapy-cookbook", 并添加你在GitHub上面的工程HTTPS链接, 选择仓库类型为Git
4. 其他项目根据自己的需要填写后点击 "Create"，创建完后会自动去激活Webhooks，不用再去GitHub设置
5. 一切搞定，从此只要你往这个仓库push代码，readthedoc上面的文档就会自动更新.

注：在创建read the docs项目时候，语言选择"Simplified Chinese"

在构建过程中出现任何问题，都可以登录readthedoc找到项目中的"构建"页查看构建历史，点击任何一条查看详细日志:
![](http://xnstatic-1253397658.cossh.myqcloud.com/rtd02.png)

我将自己以前博客里面的关于scrapy的文章都迁移至readthedoc，现在看看效果：
![](http://xnstatic-1253397658.cossh.myqcloud.com/rtd03.png)

## FAQ

build的时候出现错误：! Package inputenc Error: Unicode char 我 (U+6211)

解决办法，在`conf.py`中添加:
``` python
latex_elements={# The paper size ('letterpaper' or 'a4paper').
'papersize':'a4paper',# The font size ('10pt', '11pt' or '12pt').
'pointsize':'12pt','classoptions':',oneside','babel':'',#必須
'inputenc':'',#必須
'utf8extra':'',#必須
# Additional stuff for the LaTeX preamble.
'preamble': r"""
\usepackage{xeCJK}
\usepackage{indentfirst}
\setlength{\parindent}{2em}
\setCJKmainfont{WenQuanYi Micro Hei}
\setCJKmonofont[Scale=0.9]{WenQuanYi Micro Hei Mono}
\setCJKfamilyfont{song}{WenQuanYi Micro Hei}
\setCJKfamilyfont{sf}{WenQuanYi Micro Hei}
\XeTeXlinebreaklocale "zh"
\XeTeXlinebreakskip = 0pt plus 1pt
"""}
```


