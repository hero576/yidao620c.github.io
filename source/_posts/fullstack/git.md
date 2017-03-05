---
title: "git简明教程"
date: 2017-02-06 21:32:30 +0800
comments: true
toc: true
categories: fullstack
tags: [git]
---
很早就想些一篇关于git的文章了，这玩意儿实在好用，但是内容又比较多，
这里我讲解最基本使用技巧，这个足以应对99%以上的场景，剩下那些真的要用到就去看官网手册。

Git是目前世界上最先进的分布式版本控制系统（没有之一），它的诞生也是个很有趣的故事。
大家都知道Git是Linus大神写的，据说刚开始的时候，linux内核源码使用BitKeeper这个商业版本控制系统，
授权Linux社区免费使用，但是某一天开发Samba的Andrew这个家伙试图破解BitKeeper协议，东窗事发。
于是BitKeeper公司努力收回了免费使用权。Linus大神是不可能去道歉的，于是他就花了2个星期用C语言写了Git，
一个月内，Linux源码就由Git管理了，无敌是多么寂寞 →_→ <!--more-->

相比较像svn这样的集中式版本管理，分布式版本管理优势在哪里呢？
这里先说两个，后面再说另外几个杀手级优点。

首先，分布式所有客户机都有一个完整拷贝，所以不用担心服务器挂点。
另外分布式不需要联网就可以工作，没有中央服务器。

## 安装git
在linux上面基本就是一条命令:
``` bash
yum install git
```

如果在windows上面，就去官网下载安装文件，点击安装即可。

安装完成，还要简单配置下全局设置:
``` bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
```

## 初始化

### 初始化仓库:
``` bash
mkdir gitdemo && cd gitdemo
git init
# Initialized empty Git repository in /root/gitdemo/.git/
```
当前目录下多了一个.git的目录，这个目录是Git来跟踪管理版本库的，
没事千万不要手动修改这个目录里面的文件，不然改乱了，就把Git仓库给破坏了。

### .gitignore文件
如果你有一些文件不需要版本跟踪就写到这里面，比如:
```
build/
*.pyc
```
接受通配符，具体规则请参考[gitignore说明](https://git-scm.com/docs/gitignore)

### 添加文件到版本库
先编写一个readme.txt文件，内容如下:
```
hello git
I like git very much
```

第一步，使用`git add`命令添加read.txt到git:
``` bash
git add readme.txt
```
执行上面的命令，没有任何显示，这就对了

第二步，使用`git commit`命令提交到git仓库:
``` bash
git commit -m "readme file"
```
输出:
```
[master (root-commit) 50f5fdc] readme file
 1 file changed, 2 insertions(+)
 create mode 100644 readme.txt
```

## 回退和撤销
刚刚提交完后，继续编辑readme.txt，内容如下:
```
hello git
I like git very much
add something
```

现在，运行`git status`命令看看结果：
```
[root@controller161 gitdemo]# git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```
`git status`命令可以让我们时刻掌握仓库当前的状态，上面的命令告诉我们，readme.txt被修改过了，但还没有准备提交的修改。

虽然Git告诉我们readme.txt被修改了，但如果能看看具体修改了什么内容，自然是很好的，这时候使用`git diff`命令:
```
[root@controller161 gitdemo]# git diff readme.txt
diff --git a/readme.txt b/readme.txt
index 2236b09..4114c7f 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1,2 +1,3 @@
 hello git
 I like git very much
+add something
```
知道了对readme.txt作了什么修改后，再把它提交到仓库就放心多了，步骤还是先add，再commit:
``` bash
git add readme.txt
```
执行add之后，我们先不提交，看看状态:
```
[root@controller161 gitdemo]# git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	modified:   readme.txt
```
`git status`告诉我们，将要被提交的修改包括readme.txt，下一步，就可以放心地提交了:
``` bash
[root@controller161 gitdemo]# git commit -m "modify readme"
[master 1d57c05] modify readme
 1 file changed, 1 insertion(+)
```

提交后，我们再用git status命令看看仓库的当前状态：
``` bash
[root@controller161 gitdemo]# git status
On branch master
nothing to commit, working directory clean
```

没有需要提交的修改，而且，工作目录是干净（working directory clean）的

### 版本回退
现在再修改readme.txt文件如下:
```
hello git
I like git very much
add something
我又加了点东西在后面
```
然后再提交:
```
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "添加中文行"
[master 90507c6] 添加中文行
 1 file changed, 1 insertion(+)
```

像这样，你不断对文件进行修改，然后不断提交修改到版本库里，实际上这个commit操作就相对于一个快照。
以后你误改误删了文件是可以回退的。

我们可以通过`git log`命令查看提交历史:
```
[root@controller161 gitdemo]# git log --pretty=oneline
90507c6859f8af73a651f033de8ea901811cb4e8 添加中文行
1d57c05ee44d590f279050276884551e77d2ffb1 modify readme
50f5fdcea453fed2eee690b5e686994040ffe210 readme file
```
第一列是commit的一个id号(版本号)，是SHA1计算出来的一个非常大的数字，用十六进制表示。

假如你想回退到`modify readme`那个版本，可以通过`git reset`命令:
```
[root@controller161 gitdemo]# git reset --hard 1d57c05ee44
Unstaged changes after reset:
M	readme.txt
```
reset后面指定版本号，你可以只取前面几位，只要能区分就行。
我们再来看readme.txt:
```
[root@controller161 gitdemo]# cat readme.txt
hello git
I like git very much
add something
```
确实回退到那个提交版本了。

现在，你回退到了某个版本，关掉了电脑，第二天早上就后悔了，想恢复到新版本怎么办？找不到新版本的commit id怎么办？

在Git中，总是有后悔药可以吃的。回退必须指定commit id，而Git提供了一个命令git reflog用来记录你的每一次命令：
```
[root@controller161 gitdemo]# git reflog
1d57c05 HEAD@{0}: reset: moving to 1d57c05ee44
90507c6 HEAD@{1}: commit: 添加中文行
1d57c05 HEAD@{2}: commit: modify readme
50f5fdc HEAD@{3}: commit (initial): readme file
```

## 工作区和暂存区
在git里面有三个很重要的概念：工作区、暂存区、版本库。

### 工作区（Working Directory）
就是你在电脑里能看到的目录，比如我的gitdemo文件夹就是一个工作区

### 版本库（Repository）
工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库。
里面存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区，
还有Git为我们自动创建的第一个分支master，以及指向master的一个指针叫HEAD。

![](http://xnstatic-1253397658.cossh.myqcloud.com/git01.jpg)

把文件往Git版本库里添加的时候，是分两步执行的：

1. 第一步是用`git add`把文件添加进去，实际上就是把文件修改添加到暂存区；
2. 第二步是用`git commit`提交更改，实际上就是把暂存区的所有内容提交到当前分支。

再次修改readme.txt:
```
hello git
I like git very much
add something
这次加一行啦啦啦
```

另外再添加一个文件LICENSE，内容如下:
```
MIT
```

先用`git status`查看一下状态:
```
[root@controller161 gitdemo]# git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	LICENSE

no changes added to commit (use "git add" and/or "git commit -a")
```

Git非常清楚地告诉我们，readme.txt被修改了，而LICENSE还从来没有被添加过，所以它的状态是Untracked

现在，使用两次命令`git add`或者`git add --all`，把readme.txt和LICENSE都添加后，用`git status`再查看一下：
```
[root@controller161 gitdemo]# git add --all
[root@controller161 gitdemo]# git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   LICENSE
	modified:   readme.txt
```
现在，暂存区的状态就变成这样了:
![](http://xnstatic-1253397658.cossh.myqcloud.com/git02.jpg)

所以，`git add`命令实际上就是把要提交的所有修改放到暂存区（Stage），
然后，执行`git commit`就可以一次性把暂存区的所有修改提交到分支:
```
[root@controller161 gitdemo]# git commit -m "show stage work"
[master 32db843] show stage work
 2 files changed, 2 insertions(+)
 create mode 100644 contact.txt
```

一旦提交后，如果你又没有对工作区做任何修改，那么工作区就是“干净”的：
```
[root@controller161 gitdemo]# git status
On branch master
nothing to commit, working directory clean
```
现在版本库变成了这样，暂存区就没有任何内容了：
![](http://xnstatic-1253397658.cossh.myqcloud.com/git02.jpg)

## 关于diff
很多时候需要用diff命令来比较文件差异，总结一下:

* git diff readme.txt         -> 工作区 和 暂存区比较
* git diff --cache readme.txt -> 暂存区 和 版本库比较
* git diff HEAD -- readme.txt -> 版本库 和 工作区比较

## 撤销修改
有时候你也会犯傻修改了不该改的东西，这样时候可以通过`git checkout`命令撤销修改。

命令`git checkout -- readme.txt`意思就是，把readme.txt文件在工作区的修改全部撤销，这里有两种情况：

一种是readme.txt自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；

一种是readme.txt已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。

总之，就是让这个文件回到最近一次git commit或git add时的状态。

`git checkout -- file` 命令中的--很重要，没有--，就变成了“切换到另一个分支”的命令，
我们在后面的分支管理中会再次遇到`git checkout`命令。

还有一种情况是，你想将暂存区的修改撤销掉，重新放回工作区:
```
git reset HEAD readme.txt
```
`git reset`命令既可以回退版本，也可以把暂存区的修改回退到工作区。当我们用HEAD时，表示最新的版本。

还记得如何丢弃工作区的修改吗？
```
git checkout -- readme.txt
```
整个世界终于清静了

总结一下撤销修改场景:

1. 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`
2. 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，
第一步用命令`git reset HEAD file`，就回到了场景1，第二步按场景1操作。
3. 场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退一节，不过前提是没有推送到远程库。

## 远程仓库
可以自己搭载，也可以使用公共免费的平台，比如github，这里我就不折腾直接使用github。
关于github上面怎样注册，怎样新建仓库请参考[官网指南](https://guides.github.com/activities/hello-world/)

### 克隆远程仓库
先有远程仓库，直接通过远程仓库初始化本地仓库，比如:
``` bash
git clone https://github.com/yidao620c/scrapy-cookbook.git
```

### 添加远程库
先有本地库，后有远程库的时候，如何关联远程库呢？就使用`git remote add`命令:
```
git remote add origin https://github.com/yidao620c/scrapy-cookbook.git
```
本地做了修改commit后，再将本地仓库推送至远程:
```
git push -u origin master
```

## 分支管理
终于要介绍git的杀手级特性分支了，这也是大部分人使用git的原因。
Git的分支是与众不同的，无论创建、切换和删除分支，Git在1秒钟之内就能完成！无论你的版本库是1个文件还是1万个文件。

### 创建与合并分支
每次提交，Git都把它们串成一条时间线，这条时间线就是一个分支。截止到目前，只有一条时间线，
在Git里，这个分支叫主分支，即master分支。HEAD严格来说不是指向提交，
而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。

一开始的时候，master分支是一条线，Git用master指向最新的提交，再用HEAD指向master，就能确定当前分支，以及当前分支的提交点：
![](http://xnstatic-1253397658.cossh.myqcloud.com/git04.png)

每次提交，master分支都会向前移动一步，这样，随着你不断提交，master分支的线也越来越长。

当我们创建新的分支，例如dev时，Git新建了一个指针叫dev，
指向master相同的提交，再把HEAD指向dev，就表示当前分支在dev上：
![](http://xnstatic-1253397658.cossh.myqcloud.com/git05.png)

Git创建一个分支很快，就是增加一个dev指针，改改HEAD的指向。

不过，从现在开始，对工作区的修改和提交就是针对dev分支了，比如新提交一次后，dev指针往前移动一步，而master指针不变
![](http://xnstatic-1253397658.cossh.myqcloud.com/git06.png)

假如我们在dev上的工作完成了，就可以把dev合并到master上。Git怎么合并呢？
最简单的方法，就是直接把master指向dev的当前提交，就完成了合并：
![](http://xnstatic-1253397658.cossh.myqcloud.com/git07.png)

所以Git合并分支也很快！就改改指针，工作区内容也不变。

合并完分支后，甚至可以删除dev分支。删除dev分支就是把dev指针给删掉，删掉后，我们就剩下了一条master分支:
![](http://xnstatic-1253397658.cossh.myqcloud.com/git08.png)

下面开始实战演练一下。

首先，我们创建dev分支，然后切换到dev分支:
```
[root@controller161 gitdemo]# git checkout -b dev
Switched to a new branch 'dev'
```

`git checkout`命令加上-b参数表示创建并切换，相当于以下两条命令:
```
git branch dev
git checkout dev
```

然后用`git branch`命令查看当前分支:
```
[root@controller161 gitdemo]# git branch
* dev
  master
```

`git branch`命令会列出所有分支，当前分支前面会标一个*号。

然后，我们就可以在dev分支上正常提交，比如对readme.txt做个修改，加上一行:
```
hello git
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

然后提交：
```
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "dev commit 1"
[dev 97e84a1] dev commit 1
 1 file changed, 1 insertion(+)
```

现在，dev分支的工作完成，我们就可以切换回master分支:
```
[root@controller161 gitdemo]# git checkout master
Switched to branch 'master'
```

切换回master分支后，再查看一个readme.txt文件，刚才添加的内容不见了！
因为那个提交是在dev分支上，而master分支此刻的提交点并没有变:
![](http://xnstatic-1253397658.cossh.myqcloud.com/git08.png)

现在，我们把dev分支的工作成果合并到master分支上:
```
[root@controller161 gitdemo]# git merge dev
Updating 240afec..97e84a1
Fast-forward
 readme.txt | 1 +
 1 file changed, 1 insertion(+)
```

git merge命令用于合并指定分支到当前分支。合并后，再查看readme.txt的内容，就可以看到，和dev分支的最新提交是完全一样的

合并完成后，就可以放心地删除dev分支了:
```
[root@controller161 gitdemo]# git branch -d dev
Deleted branch dev (was 97e84a1)
```

删除后，查看branch，就只剩下master分支了:
```
[root@controller161 gitdemo]# git branch
* master
```

因为创建、合并和删除分支非常快，所以Git鼓励你使用分支完成某个任务，
合并后再删掉分支，这和直接在master分支上工作效果是一样的，但过程更安全。

## 解决冲突
冲突经常会发生，我们在不同分支上面修改了同一个文件，最后合并的时候肯定会有冲突。现在我们演示下冲突。

先开一个feature1分支:
```
[root@controller161 gitdemo]# git checkout -b feature1
Switched to a new branch 'feature1'
```

在feature1分支上面修改readme.txt如下:
```
hello git, feature1
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

然后提交:
```
[root@controller161 gitdemo]# git add --all
[root@controller161 gitdemo]# git commit -m "feature1 111"
[feature1 f0e23f8] feature1 111
 1 file changed, 1 insertion(+), 1 deletion(-)
```

切换到master分支，修改同样的文件readme.txt,内容如下:
```
hello git master
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

然后也提交:
```
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "master"
[master 18c3eb0] master
 1 file changed, 1 insertion(+), 1 deletion(-)
```

现在，master分支和feature1分支各自都分别有新的提交，变成了这样:
![](http://xnstatic-1253397658.cossh.myqcloud.com/git10.png)

这种情况下，Git无法执行“快速合并”，试试合并看看:
```
[root@controller161 gitdemo]# git merge feature1
Auto-merging readme.txt
CONFLICT (content): Merge conflict in readme.txt
Automatic merge failed; fix conflicts and then commit the result.
```

readme.txt文件存在冲突，必须手动解决冲突后再提交。`git status`也可以告诉我们冲突的文件:
```
[root@controller161 gitdemo]# git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")

Unmerged paths:
  (use "git add <file>..." to mark resolution)

	both modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

打开readme.txt，内容变成这样:
```
<<<<<<< HEAD
hello git master
=======
hello git, feature1
>>>>>>> feature1
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

Git用<<<<<<<，=======，>>>>>>>标记出不同分支的内容，这是典型的diff冲突格式，我们修改如下后保存:
```
hello git master and feature1
I like git very much
add something
这次加一行啦啦啦
this is dev line
```
再提交:
```
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m 'fix confict'
[master a2c9e32] fix confict
```

现在，master分支和feature1分支变成了下图所示:
![](http://xnstatic-1253397658.cossh.myqcloud.com/git11.png)

查看合并图:
``` bash
[root@controller161 gitdemo]# git log --graph --pretty=oneline --abbrev-commit
*   a2c9e32 fix confict
|\
| * f0e23f8 feature1 111
* | 18c3eb0 master
|/
* 97e84a1 dev commit 1
* 240afec show stage work
* 32db843 show stage work
* 1d57c05 modify readme
* 50f5fdc readme file
```

如果想保留某个分支的提交记录，删除后也有记录历史。可禁用Fast forward:
```
git merge --no-ff -m "merge with no-ff" dev
```

本次合并要创建一个新的commit，所以加上-m参数，把commit描述写进去。

## 分支策略
在实际开发中，我们应该按照几个基本原则进行分支管理:

首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，
再把dev分支合并到master上，在master分支发布1.0版本；

你和你的小伙伴们每个人都在dev分支上干活，每个人都有自己的分支，时不时地往dev分支上合并就可以了，类似这样:
![](http://xnstatic-1253397658.cossh.myqcloud.com/git12.png)

## Bug分支
bug分支一般是那种时间短任务急的分支，修改完马上要合并提交的，就是临时分支。
但是你一般都在dev分支上面工作，这时候有些工作还没有提交。
```
[root@controller161 gitdemo]# vim readme.txt
[root@controller161 gitdemo]# git branch
* dev
  master
[root@controller161 gitdemo]# git status
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

并不是你不想提交，而是工作只进行到一半，还没法提交，预计完成还需1天时间。但是，必须在两个小时内修复该bug，怎么办？

幸好，Git还提供了一个stash功能，可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作:
```
[root@controller161 gitdemo]# git stash
Saved working directory and index state WIP on dev: a2c9e32 fix confict
HEAD is now at a2c9e32 fix confict
```

可以放心地创建分支来修复bug，假定需要在master分支上修复，就从master创建临时分支:
```
[root@controller161 gitdemo]# git checkout master
Switched to branch 'master'
[root@controller161 gitdemo]# git checkout -b issue-101
Switched to a new branch 'issue-101'
```

现在修复bug，readme.txt修改为:
```
hello git master and feature1
I like git very much
add something, fix bug101
这次加一行啦啦啦
this is dev line
```

然后提交:
```
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "fix bug101"
[issue-101 a4401b2] fix bug101
 1 file changed, 1 insertion(+), 1 deletion(-)
```

修复完成后，切换到master分支，并完成合并，最后删除issue-101分支:
```
[root@controller161 gitdemo]# git merge --no-ff -m "merged bug fix 101" issue-101
Merge made by the 'recursive' strategy.
 readme.txt | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

改完bug，是时候接着回到dev分支干活了:
```
[root@controller161 gitdemo]# git checkout dev
Switched to branch 'dev'
[root@controller161 gitdemo]# git status
On branch dev
nothing to commit, working directory clean
```

切换回来后第一件事就是把刚刚的bug分支合并:
```
git merge --no-ff -m "dev-merge-m" master
```

工作区是干净的，刚才的工作现场存到哪去了？用git stash list命令看看:
```
[root@controller161 gitdemo]# git stash list
stash@{0}: WIP on dev: a2c9e32 fix confict
```

`git stash pop`，恢复的同时把stash内容也删了（如果提示冲突，去文件手动改正）:
```
[root@controller161 gitdemo]# git stash pop
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (2df611a386511cc7989db79457b620b9971c08b5)
```

你可以多次stash，恢复的时候，先用git stash list查看，然后恢复指定的stash，用命令:
```
git stash apply stash@{0}
```

有必要再次总结一下。

master合并merge解决好的bug后，不要先把dev解印，先合并master，获取里面的bug方案后，在解印。解印时会有提示冲突，需手动改一次文件。

1. 在  dev 下正常开发中，说有1个bug要解决，首先我需要把dev分支封存stash
2. 在master下新建一个issue-101分支，解决bug，成功后
3. 在master下合并issue-101
4. 在 dev  下合并master，  这样才同步了里面的bug解决方案
5. 解开dev封印stash pop，系统自动合并 & 提示有冲突，因为封存前dev写了东西，此时去文件里手动改冲突
6. 继续开发dev，最后add，commit
7. 在master下合并最后完成的dev

代码过程如下：
```
1. $ git stash
2. $ git checkout master
   $ git checkout -b issue-101
   //去文件里修bug
   $ git add README.md
   $ git commit -m "fix-issue-101"
3. $ git checkout master
   $ git merge --no-ff -m "m-merge-issue-101" issue-101
   $ git branch -d issue-101
4. $ git checkout dev
   $ git merge --no-ff -m "dev-merge-m" master
5. $ git stash pop
            //提示冲突，去文件手动改正
            Auto-merging README.md
            CONFLICT (content): Merge conflict in README.md
6. //继续开发 ... ... ，完成后一并提交
   $ git add README.md
   $ git commit -m "fixconflict & append something"
7. $ git checkout master
   $ git merge --no-ff -m "m-merge-dev" dev
   $ git branch -d dev
```

开发一个新feature，最好新建一个分支。

如果要丢弃一个没有被合并过的分支，可以通过git branch -D <name>强行删除。

## 多人协作

* 查看远程库信息，使用`git remote -v`；
* 本地新建的分支如果不推送到远程，对其他人就是不可见的；
* 从本地推送分支，使用`git push origin branch-name`，如果推送失败，先用git pull抓取远程的新提交；
* 在本地创建和远程分支对应的分支，使用`git checkout -b branch-name origin/branch-name`，本地和远程分支的名称最好一致；
* 建立本地分支和远程分支的关联，使用`git branch --set-upstream branch-name origin/branch-name`；
* 从远程抓取分支，使用`git pull`，如果有冲突，要先处理冲突。

## 标签管理

* 命令`git tag <name>`用于新建一个标签，默认为HEAD，也可以指定一个commit id；
* 命令`git tag -a <tagname> -m "blablabla..."`可以指定标签信息；
* 命令`git tag -s <tagname> -m "blablabla..."`可以用PGP签名标签；
* 命令`git tag`可以查看所有标签。
* 命令`git push origin <tagname>`可以推送一个本地标签；
* 命令`git push origin --tags`可以推送全部未推送过的本地标签；
* 命令`git tag -d <tagname>`可以删除一个本地标签；
* 命令`git push origin :refs/tags/<tagname>`可以删除一个远程标签。

## 使用GitHub

在GitHub出现以前，开源项目开源容易，但让广大人民群众参与进来比较困难，也只能把diff文件用邮件发过去，很不方便。

但是在GitHub上，利用Git极其强大的克隆和分支功能，广大人民群众真正可以第一次自由参与各种开源项目了。

* 在GitHub上，可以任意Fork开源仓库；
* 自己拥有Fork后的仓库的读写权限；
* 可以推送pull request给官方仓库来贡献代码。

关于GitHub的使用比较简单，参考[官方指南](https://guides.github.com/activities/hello-world/)，
大概10分钟就能看懂整个流程，就不多说了。

