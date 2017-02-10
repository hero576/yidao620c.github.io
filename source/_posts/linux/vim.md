---
layout: post
title: "vim笔记"
date: 2016-06-02 20:07:22 +0800
toc: true
categories: linux
tags: [vim]
---

vim 是 Linux 系统上的最著名的文本/代码编辑器，也是早年的 Vi 编辑器的加强版，
而 gvim 则是其 Windows 版。它的最大特色是完全使用键盘命令进行编辑，脱离了鼠标操作虽然使得入门变得困难，
但上手之后键盘流的各种巧妙组合操作却能带来极为大幅的效率提升。

因此 vim 和现代的编辑器（如 Sublime Text）有着非常巨大的差异，而且入门学习曲线陡峭，
需要记住很多按键组合和命令，如今被看作是高手、Geek们专用的编辑器。尽管 vim 已经是古董级的软件，
但还是有无数新人迎着困难去学习使用，可见其经典与受欢迎程度。另外，由于 vim 的可配置性非常强，
各种插件、语法高亮配色方案等多不胜数，无论作为代码编辑器或是文稿撰写工具都非常给力。<!--more-->

## 安装
这一步比较简单，在centos上面一条命令:
``` bash
yum install vim
```

## 配置
我这里有一个比较优化的配置了，`vim ~/.vimrc`，输入如下内容
``` bash
"这个文件的双引号 (") 是注释
syntax on               "语法高亮显示。
set hlsearch            "高亮度反白
set backspace=2         "可以用Backspace键删除
set ts=4                "tab键等于4个空格
set expandtab           "tab键自动变空格
set tabstop=4
set softtabstop=4
set autoindent          "自动缩进
set pastetoggle=<F9>    "插入模式粘贴按F9取消自动缩进
set ruler               "可显示最后一行的状态
set showmode            "左下角那一行的状态
set nu                  "可以在每一行的最前面显示行号啦！
set bg=dark             "显示不同的底色色调
"set cursorline          "光标所在行一横线
set laststatus=2        "显示当前编辑文件名
set showcmd
set magic
set lazyredraw
set history=100         "历史记录条数
set hlsearch            "高亮显示搜索结果
set incsearch           "增量搜索，每次输入一个字母都自动搜
vnoremap ,s y:%s/<C-R>=escape(@", '\\/.*$^~[]')<CR>/    "选中替换
```

## 入门

vim有三种模式：

1. 一般模式：在Linux终端中输入“vim 文件名”就进入了一般模式,但不能输入文字。
2. 编辑模式：在一般模式下按i就会进入编辑模式，此时就可以写程式，按Esc可回到一般模式。
3. 命令模式：在一般模式下按：就会进入命令模式，左下角会有一个冒号出现，此时可以敲入命令并执行。

当你安装好一个编辑器后，你一定会想在其中输入点什么东西，然后看看这个编辑器是什么样子。
但vim不是这样的，请按照下面的命令操作:

* 启动Vim后，vim在 Normal 模式下。
* 让我们进入 Insert 模式，请按下键 i 。(vim左下角有一个–insert–字样，表示你可以输入了）
* 此时，你可以输入文本了，就像你用“记事本”一样。
* 如果你想返回 Normal 模式，请按 ESC 键

现在，你知道如何在 Insert 和 Normal 模式下切换了。下面是一些命令，可以让你在 Normal 模式下幸存下来：
```
i   → Insert 模式，按 ESC 回到 Normal 模式.
x   → 删当前光标所在的一个字符。
:wq → 存盘 + 退出 (:w 存盘, :q 退出)   （陈皓注：:w 后可以跟文件名）
dd  → 删除当前行，并把删除的行存到剪贴板里
p   → 粘贴剪贴板
```

推荐:
```
hjkl (强例推荐使用其移动光标，但不必需) →你也可以使用光标键 (←↓↑→). 注: j 就像下箭头。
:help <command> → 显示相关命令的帮助。你也可以就输入 :help 而不跟命令。（陈皓注：退出帮助需要输入:q）
```

## 进阶1
现在是时候学习一些更多的命令了，下面是我的建议：

各种插入模式
```
a  → 在光标后插入
o  → 在当前行后插入一个新行
O  → 在当前行前插入一个新行
cw → 替换从光标所在位置后到一个单词结尾的字符
```

简单的移动光标
```
0         → 数字零，到行头
^         → 到本行第一个不是blank字符的位置（所谓blank字符就是空格，tab，换行，回车等）
$         → 到本行行尾
g_        → 到本行最后一个不是blank字符的位置。
/pattern  → 搜索 pattern 的字符串（陈皓注：如果搜索出多个匹配，可按n键到下一个）
```

拷贝/粘贴(p/P都可以，p是表示在当前位置之后，P表示在当前位置之前)
```
P  → 粘贴
yy → 拷贝当前行当行于 ddP
```

Undo/Redo
```
u     → undo
<C-r> → redo
```

打开/保存/退出/改变文件(Buffer)
```
:e <path/to/file>      → 打开一个文件
:w                     → 存盘
:saveas <path/to/file> → 另存为 <path/to/file>
:x， ZZ 或 :wq          → 保存并退出 (:x 表示仅在需要时保存，ZZ不需要输入冒号并回车)
:q!                    → 退出不保存 :qa! 强行退出所有的正在编辑的文件，就算别的文件有更改。
:bn 和 :bp             → 你可以同时打开很多文件，使用这两个命令来切换下一个或上一个文件。
```

## 进阶2
重复命令次数
```
N<command> → 重复某个命令N次
2dd        → 删除2行
3p         → 粘贴文本3次
```

你要让你的光标移动更有效率，你一定要了解下面的这些命令，千万别跳过
```
NG → 到第 N 行 （陈皓注：注意命令中的G是大写的，另我一般使用 : N 到第N行，如 :137 到第137行）
gg → 到第一行。（陈皓注：相当于1G，或 :1）
G  → 到最后一行
```
按单词移动
```
w      → 到下一个单词的开头。
e      → 到下一个单词的结尾。
```

下面这三个命令对程序员来说是相当强大的
```
%      → 匹配括号移动，包括 (, {, [. （你需要把光标先移到括号上）
* 和 #  →  匹配光标当前所在的单词，移动光标到下一个（或上一个）匹配单词（*是下一个，#是上一个）
```

很多命令都是如下方式工作：
```
<start position><command><end position>
```

例如`0y$`命令意味着：
```
0 → 先到行头
y → 从这里开始拷贝
$ → 拷贝到本行最后一个字符
```

还有很多时间并不一定你就一定要按y才会拷贝，下面的命令也会被拷贝
```
d  (删除 )
v  (可视化的选择)
gU (变大写)
gu (变小写)
```

## 进阶3
你只需要掌握前面的命令，你就可以很舒服的使用VIM了。但是，现在，我们向你介绍的是VIM杀手级的功能

在当前行上移动光标:
```
0      → 到行头
^      → 到本行的第一个非blank字符
$      → 到行尾
g_     → 到本行最后一个不是blank字符的位置。
fa     → 到下一个为a的字符处，你也可以fs到下一个为s的字符。
t,     → 到逗号前的第一个字符。逗号可以变成其它字符。
3fa    → 在当前行查找第三个出现的a。
F 和 T  → 和 f 和 t 一样，只不过是相反方向。
```
## 区块操作
典型列编辑模式：
```
<Ctrl-v>  → 开始块操作
<hjkl>    → 移动光标
<Shift-i> → 插入字符
<ESC>     → 生效
```

可视化选择： v,V,<C-v>

## 多tab编辑
在不同的tab中打开多个文档，这个特性我一直很喜欢，就跟浏览器或各个IDE中是一样的
```
:tabnew filename在一个新的tab中打开文件
gt在不同的tab中切换，use 5gt to switch to tab 5，从1开始
:tabclose或者:q关闭当前tab
:tabmove将tab移动到指定的位置

:tabedit {file}   edit specified file in a new tab
:tabclose         close current tab
:tabclose {i}     close i-th tab
:tabonly          close all other tabs
:tabs         list all tabs
:tabm 0       move current tab to first
:tabm         move current tab to last
:tabm {i}     move current tab to position i+1
:tabn         go to next tab
:tabp         go to previous tab
:tabfirst     go to first tab
:tablast      go to last tab
```

## FAQ
不小心按到ctrl+s，结果就不动了

原来是Linux的一个快捷键呀，干什么用的？
原来Ctrl+S在Linux里，是锁定屏幕的快捷键。如果要解锁，按下Ctrl+Q就可以了



