---
title: "Gitea 安装使用"
date: 2021-03-22T10:07:35+08:00
draft: false
description: 安装 Gitea 创建自己的git服务
tags: ["git"]
keywords: ["git"]
categories: ["分享"]
isCJKLanguage: true
enableDisqus: true
---

## 前言

今天偶然看到一个新的开源的git服务软件[Gitea](https://gitea.io/)，一看到界面，瞬间就爱了，因为之前我自己用的是[gitblit](http://gitblit.github.io/gitblit/)，界面比较简单，主要是用来管理公司的一些小项目。今天看到Gitea之后，就决定迁移到过去，简单折腾了一下，配置起来比gitblit要简单一些，但界面却更加漂亮了，总体上看起来比较像github，并且还支持主题系统，很合我的胃口。

## 下载二进制包

首先去下载对应系统的二进制包，可以去[github](https://github.com/go-gitea/gitea/releases)或者[官网](https://dl.gitea.io/gitea)下载最新的发布版本。

我是在windows下配置的，所以选择下载windows版的可执行程序。

## 开启服务

下载后不需要安装，直接就能运行，但直接运行的话会有一个控制台显示在桌面，所以可以考虑将程序作为一个系统服务在后台运行。

1. 由于不需要安装，可以直接将下载的可执行程序放在自己想要安装的目录，eg: `D:\gitea\gitea.exe`

2. 以**管理员方式**打开`cmd`或者`powershell`，执行命令：

    ``` sh
    sc create gitea start= auto binPath= "\"D:\gitea\gitea.exe\" web --config \"D:\gitea\custom\conf\app.ini\""
    ```

3. 打开系统的服务管理界面，找到`gitea`，此时是**已停止**状态，按下鼠标右键，在弹出的菜单选择**开始**，然后可以看到状态变为**正在启动**。

4. 打开浏览器访问 [http://localhost:3000](http://localhost:3000)，应该就能看到Gitea的界面了，点击页面上的**探索**（explore），还需要进行一些配置才能正常运行，数据库可以直接使用`sqlite`，这样就不需要再安装其它的数据库了，然后就是一些目录配置，根据需要选择目录就行。

5. 最后可以创建管理员账号，也可以不用设置，完成安装后第一个注册的账号会自动成为管理员。

到这里安装和配置就完成了，此时再看服务中的`gitea`的状态，已经变为**正在运行**。

## 迁移仓库到 Gitea

Gitea 自带**迁移外部仓库**的功能，但我一直导入失败，提示**没有导入本地仓库的权限**。后来只能将本地的仓库推送到Gitea，不过效果是一样的。

1. 首先在Gitea中新建一个同名的仓库，复制仓库的git链接。

2. 修改本地仓库的`origin`

    ``` sh
    $ git remote origin set-url http://localhost:3000/user/myrepo.git
    ```

    或者直接修改`.git/config`文件中的`origin` url。

3. 推送本地仓库到Gitea

    ``` sh
    $ git push origin main
    ```

    静静等待推送完成，然后刷新gitea的仓库页面，就可以看到所有的提交记录了。

## 启用 SSH

编辑 `D:\gitea\custom\conf\app.ini`，在`[server]`部分新增一项配置

``` ini
[server]
START_SSH_SERVER = true
```

保存后，在服务中重启 gitea 即可。

在gitea的账号管理中新增SSH密钥，然后就可以使用ssh的方式管理账户所拥有的仓库了。

## 最后

总的来看，Gitea的配置十分简单，基本上下载后就可以使用，没有其它的依赖，gitblit则需要安装java运行环境，界面十分漂亮，功能也比较完善，自用完全没有问题。
