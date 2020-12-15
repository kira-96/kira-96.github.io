---
title: 一些常用的git命令
date: 2020-11-26T09:45:36+08:00
draft: false
description: 一些常用的git命令
tags: [ "git" ]
keywords: [ "git" ]
categories: [ "分享" ]
isCJKLanguage: true
---

## 说明

这里是个人工作时常用的一些git命令，现在越来越多了，小本本都快记不下了，这里稍微做一下整理。

## 工具下载

首先是git的下载地址：

- 官网：[https://git-scm.com/](https://git-scm.com/)
- taobao镜像：[https://npm.taobao.org/mirrors/git-for-windows/](https://npm.taobao.org/mirrors/git-for-windows/)

由于官网的下载速度很慢，推荐使用taobao镜像的下载地址。

## 设置用户名和邮箱

全局配置

``` bash
$ git config --global user.name [name]
$ git config --global user.email [email]
```

配置当前仓库

``` bash
$ git config user.name [name]
$ git config user.email [email]
```

查看用户名和邮箱

``` bash
$ git config user.name
$ git config user.email
```

## 生成SSH密钥

``` bash
$ ssh-keygen -t rsa -C "[email]"
```

执行完毕和用户目录下就会生成一个**.ssh**文件夹。

## 拉取远程代码库

可以直接`clone`远程代码仓库（推荐）

``` bash
$ git clone https://github.com/libgit2/libgit2 mylibgit
```

先初始化仓库，再拉取代码

``` bash
$ cd ./mylibgit
$ git init
$ git remote add origin git@github.com:libgit2/libgit2.git
$ git pull origin master
$ git push -u origin master
```

## 提交代码变更

查看工作区变更

``` bash
$ git status
```

添加文件到暂存区

``` bash
$ git add README.md
```

添加所有变更文件到暂存区

``` bash
$ git add .
```

提交到本地仓库

``` bash
$ git commit -m 'Update README'
```

推送到远程代码仓库（`main`是分支）

``` bash
$ git push origin main
```

## 版本切换

查看版本/提交记录

``` bash
$ git log
```

查看版本/提交简介

``` bash
$ git log --pertty=oneline
```

### 撤销提交，保留代码变更

撤销上次提交

``` bash
$ git reset --soft HEAD^
```

撤销前n次提交

``` bash
$ git reset --soft HEAD~n
```

回退到某次提交记录

``` bash
$ git reset --soft commit-id
```

### 版本回退，不保留代码变更

回退到当前最新提交

``` bash
$ git reset --hard HEAD
```

回退到上一版本

``` bash
$ git reset --hard HEAD^
```

回退到之前的第n个版本

``` bash
$ git reset --hard HEAD~n
```

回退到某个版本/重新切换回未来版本

``` bash
$ git reset --hard commit-id
```

强制推送：在已经推送到远程的记录又被修改的情况下

``` bash
$ git push origin main --force
```

``` bash
$ git push -f origin main
```

## 分支管理

查看当前仓库分支

``` bash
$ git branch
```

查看当前仓库以及远程仓库所有分支

``` bash
$ git branch -a
```

在当前分支的基础上创建新分支

``` bash
$ git checkout -b 分支名
```

删除已合并的本地分支

``` bash
$ git branch -d 分支名
```

删除未合并的本地分支

``` bash
$ git branch -D 分支名
```

删除远程仓库分支

``` bash
$ git push origin -d 分支名
```

或

``` bash
$ git push origin :分支名
```

**删除远程已经删除的分支**

``` bash
$ git remote prune origin
```

## 标签-tag

tag和分支操作类似

查看tag

``` bash
$ git tag
```

给当前版本添加tag

``` bash
$ git tag 标签名
```

给某一版本添加tag

``` bash
$ git tag 标签名 commit-id
```

删除标签

``` bash
$ git tag -d 标签名
```

删除远程标签

``` bash
$ git push origin -d 标签名
```

推送标签到远程仓库

``` bash
$ git push origin 标签名
```

## 保持fork之后的仓库和上游同步

遇到一些好的代码仓库，有时候会fork一份到自己的账号，但一旦原来的代码仓库有了新的提交，如何保持自己的仓库和源仓库代码同步呢？

一种方式是通过远程仓库向fork之后的仓库提交一个PR，但这样会导致提交记录不一致，非常EP，在网上搜罗了好久，终于找到一个完美的方法。

首先需要将fork之后的仓库clone到本地。

然后设置本地仓库的上游仓库地址为源仓库

``` bash
$ git remote add upstream git@github.com:kira-96/myblog.git
```

同步上游仓库变更

``` bash
$ git fetch upstream
$ git checkout main
$ git merge upstream/main
```

推送到远程仓库

``` bash
$ git push origin main
```

**参考**

- [git思维导图](https://www.processon.com/view/link/5c6e2755e4b03334b523ffc3#map)
- [保持fork之后的项目和上游同步](https://github.com/staticblog/wiki/wiki/%E4%BF%9D%E6%8C%81fork%E4%B9%8B%E5%90%8E%E7%9A%84%E9%A1%B9%E7%9B%AE%E5%92%8C%E4%B8%8A%E6%B8%B8%E5%90%8C%E6%AD%A5)
