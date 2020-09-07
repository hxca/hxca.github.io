---
title: 基于 OpenWrt 的 Git 服务搭建
categories: 
- 技术经验总结
- Linux
tags:
  - Linux
  - OpenWrt
  - Git
abbrlink: a6e6b99
date: 2020-02-20 14:10:58
---

## 前言
由于疫情的缘故，公司决定分一部分员工上班，一部分员工在家工作。而我的上级领导安排我们两人一组共两组轮流上班，实际上我们组就我一个人，也就是说我是一个人值班。不过话又说回来，这样隔天上班的方式，今天在公司写代码，明天在家写代码。代码复制来复制去的，相当麻烦。本来打算托管在 GitHub 上面的，奈何访问速度实在是感人，遂放弃。而至于国内的代码托管平台，访问速度虽然很快，但是对其印象不好，所以也不考虑了。后来想了想，我自己家庭网路就有公网 IP 地址，还有动态域名。所以干脆在自己的 OpenWrt 上面部署 Git 服务。

<!-- more -->

## 搭建过程

### 安装 Git
官方 OpenWrt 默认是没有 Git 的，可以通过 `opkg` 来安装
```shell
# opkg update
# opkg install git
```

不过我的 OpenWrt 是自己编译的，在编译选参数的时候就顺便勾选 Git 了，所以不用执行。

### 创建 Git 用户
首先创建专门给 Git 服务器使用的 git 用户。由于我编译 OpenWrt 的时候，没有勾选 `useradd` 命令。所以需要自己手动修改 `/etc/passwd` ， `/etc/shadow` ， `/etc/group` 文件来添加 git 用户和用户组。简单地说一下这几个文件的作用， `/etc/passwd` 是用来定义 Linux 用户， `/etc/shadow` 存放对应用户的密码， `/etc/group` 定义用户所属的用户组。

| File          | PurPose |
| ---           | --- |
| `/etc/passwd` |　	Secure user account information |
| `/etc/shadow` |　	User account information |
| `/etc/group`  |　	Defines the groups to which users belong |

***来自 [ArchWiki - Users and groups](!https://wiki.archlinux.org/index.php/Users_and_groups#File_list)***

好了，不说那么多了，开始动手吧。首先修改 `/etc/passwd` ， 在文本末尾添加
```
git:x:700:700:git:/opt/home/git:/usr/bin/git-shell
```

修改 `/etc/shadow` ， 在文本末尾添加
```
git:x:0:0:99999:7:::
```

修改 `/etc/group` ， 在文本末尾添加
```
git:x:700:git
```

上面具体参数我就不解释了，我在文末会添加链接，遇到不懂的上去 ArchWiki 看一下吧。（我太懒了）

### 配置过程

配置 git-shell ，编辑 `/etc/shells` 文件，添加内容
```
/usr/bin/git-shell
```

添加 ssh 公钥
```shell
# mkdir .ssh
# cat "XXXXXX" >> .ssh/authorizd_keys
```

这里我们用的是其他用户创建的，所以修改 `authorized_keys` 和 `.ssh` 拥有者为 `git`	 。
```shell
# chown -R git:git .ssh
```

## 使用问题
### 关于使用非 22 端口 git clone 项目
Git 默认使用 ssh 的端口，而 ssh 默认端口是22。也就是说，Git 使用的是 22 端口。在默认情况下，在服务端使用 `git init --bare xxx.git` 新建仓库后，就可以在本地端使用 `git clone git@ip:/path/xxx.git` 复制仓库了。但是如果 ssh 设定了非 22 端口，只能这样使用 `git clone` 了
```shell
$ git clone ssh://git@hostname:port/.../xxx.git	# port 为 ssh 端口
```

例如：
```shell
$ git clone ssh://git@10.137.20.113:2222/root/test.git   # 2222 是 ssh 端口
```

### 关于新建仓库问题

由于 git 用户不能用 bash shell 登录，所以创建仓库时，需要用其他账号 ssh 上去，然后执行 `git init --bare xxx.git` 。但由于用其他账号创建的仓库所有不属于 git 用户所有。所以，又要使用 `chown -R git:git xxx.git` 来修改拥有者权限。而且每次创建仓库都要执行这条命令。相当地麻烦。于是我写了个脚本，用来执行这两条命令。并且当执行 `newrepo aab bbc` 时，同时创建 `aab` `bbc` 两个仓库，这样节省了不少事。好了不多说，直接上脚本吧。

```shell
#!/bin/bash

if [ $# == 0 ]
then
	echo "Input args"
	exit 1
fi

repo_path="/opt/home/git/"
repo_ext=".git"

for loop in $@
do
	repo=$repo_path$loop$repo_ext
	git init --bare $repo
	chown -R git:git $repo
done
```

## 参考资料

1. [Users and groups - ArchWiki](https://wiki.archlinux.org/index.php/users_and_groups)
2. [Git on the Server - Setting Up the Server](https://git-scm.com/book/en/v2/Git-on-the-Server-Setting-Up-the-Server)
3. [处理 git clone 命令的非标准SSH端口连接](https://nanxiao.me/git-clone-ssh-non-22-port/)

