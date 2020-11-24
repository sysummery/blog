---
title: Git进阶
date: 2019-12-01 12:18:22
tags:
   - git
photos:
   - ["http://sysummerblog.club/git_cover.jpeg"]
---
以下是我在工作中总结的关于git命令的一些使用方法。比较常用的那些我都不说了，主要是一些经常用但是又经常忘的命令，因此在这里总结一下，仅供参考。
<!--more-->
### 超看操作日志
查看当前分支的所有的提交日志
```sh
git log
```
查看当前分支最后n次的提交日志
```sh
git log -n
```
在一行内显示当前分支的所有提交日志
```sh
git log --oneline
```

在提交的msg里搜索当前分支的日志
```sh
git log --grep='cps'
```

根据作者搜索当前分支的日志
```sh
git log --author='suyang'
```

查看所有的提交日志，包括已经删除的分支的
```sh
git reflog
```
`git reflog`与`git log`一样也可以使用`-n, --online,--grep,--author`

### 操作子模块

克隆项目的时候连同子模块一起克隆
```sh
git clone --recursive [url][path]
```
如果克隆的时候没有连同子模块一起克隆那么还可以
```sh
git submodule update --init --recursive
```
之后想更新子模块的内容
```sh
git submodule update
```
如果想修改子模块，操作和正常的git仓库一致。

### 提交
下面的命令
```sh
git commit -m'提交' -a
```
相当于两条命令
```sh
git add .
git commit -m'提交'
```

修改上次提交的message
```sh
git commit --amend
```
如果分支已经push了，那么千万不要使用该命令。因为该命令会改变commit的hash值，这样就使当前的分支与远程的分支“开叉”了。

### 分支
```sh
git checkout -b dev origin/dev
```
这个命令的初衷是新建本地的dev分支并跟踪远程的dev分支。这个命令的执行前提是本地知道远程有个dev分支，否则会报错。那么怎么才能让本地知道远程的分支呢?使用`git pull`。这个命令不仅能拉取跟踪分支的最新代码，还能拉取分支和标签信息。

如果误删除了分支怎么办？像我这种不想留不再使用的东西的人，已经误删过好几次了，解决办法很简单先试用`git reflog`找到你误删除的分支的最后一次commit的hash值，然后通过下面的命令恢复
```sh
git checkout -b recovery hash值
```

查看远程和本地的所有分支
```sh
git branch -a
```

切换到刚才的分支
```sh
git checkout -
```
下面说说rebase。
1. 我的分支a是基于本地的master创建的，我的master跟踪远程的master
2. 我在分支a上开发，有3个commit
3. 在我开发的过程中，别人向远程的master提交了新的代码，这次提交总公有2个commit

这个时候我们可以知道本地的master和现在远程最新的master都是基于之前旧版本的master的，如果我现在要向远程推送代码那么我应该

1. 先checkout到我本地的master
2. 拉取远程最新的master到我的master
3. 在我本地merge分支a，其实在这个merge的过程就是三方合并，这三方分别是最初的master，现在远程最新的master，我的分支a

上面的这种方式会让提交看起来很难看，因此还有另外一种方法

1. 先checkout到我本地的master
2. 拉取远程最新的master到我的master
3. 切换会分支a，然后执行
```sh
git rebase master
```
这个命令的意思是，使用现在的master的分支作为基础，重新把我在a分支上的提交依次覆盖到maste分支上作为新的a的分支。变基后a分支就在当前master分支的“直接上游”
4. 切回master分支然后merge刚才变完基的a分支，这个过程是一个fast-forward

### 撤销命令汇总

撤销最后n次commit，并把撤销出来的更改放到暂存区。工作区不会受影响
```sh
git reset --soft HEAD~n
```

如果想直接丢弃撤销出来的内容
```sh
git reset --hard HEAD~n
```
将fileName从暂存区删除，这不会对此时的工作区有任何的影响，仅仅是从暂存区删除了fileName而已.但是此时git会重新对filename就行评估，如果此时filename与版本库不一致，那么filename还是会出现在工作区，但这与reset命令没有关系
```sh
git reset fileName
```

用版本库里的代码强行刷新暂存区与工作区
```sh
git checkout -f
```

拉区版本库的 fileName 同时替换工作区与暂存区的fileName
```sh
git checkout HEAD filename
```

将工作区的fileName替换为暂存区的fileName，如果暂存区没有，那么会去版本库拉去fileName替换工作区的fileName。之后filename肯定不会出现在工作区
```sh
git checkout filename
```

最后说一下git revert命令
```
git revert hash值
```
revert命令会撤销某一次的提交，与reset不同的是，当前的版本库不会回退,而是向前一步
