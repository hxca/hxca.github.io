---
title: Samba 服务器搭建
categories: 
- 技术经验总结
- Linux
tags:
  - Samba
  - Linux
abbrlink: b653765d
date: 2020-02-18 13:14:10
---

## 前言
> Samba，是种用来让 UNIX 系列的操作系统与微软 Windows 操作系统的 SMB/CIFS（Server Message Block/Common Internet File System） 网络协议做链接的自由软件。第三版不仅可访问及分享 SMB 的文件夹及打印机，本身还可以集成入 Windows Server 的网域，扮演为网域控制站（Domain Controller）以及加入 Active Directory 成员。简而言之，此软件在 Windows 与 UNIX 系列操作系统之间搭起一座桥梁，让两者的资源可互通有无。

以上摘自 [Wikipedia](https://zh.wikipedia.org/wiki/Samba) 。总的来说，在局域网中， Samba 是用来当作共享盘的。

<!-- more -->

## 搭建过程

### 安装 Samba
Ubuntu/Debian：
```shell
$ sudo apt-get install samba
```

CentOS：
```shell
$ sudo yum install samba
```


### 配置 smb.conf
首先备份 `smb.cof`
```shell
$ sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

然后修改 `smb.conf` ，在最后面添加如下内容：
```
[smbshare]
comment = smbshare home directory
path = /extdisk/disk1/smbshare
browseable = yes
public = no
writeable = yes
valid users = usmb
create mask = 0644
directory mask = 0755
force user = usmb
force group = usmb
available = yes
unix charset = UTF-8
dos charset = cp936
```

其中参数解析如下：

|   参数            |   解析                            |
|:-----------------:|:---------------------------------:|
|   public          |   设置是否允许匿名访问            |
|   path            |   设置共享文件夹的路径            |
|   valid users     |   设置允许登陆的用户名            |
|   force user      |   设置强制设定新建文件所属用户    |
|   force group     |   设置强制设定新建文件所属用户组  |
|   create mask     |   设置创建文件设定的权限          |
|   directory mask  |   设置创建文件夹设定的权限        |

**security 是设置 samba 用户认证模式。这里没有设置 security 参数是因为其默认值为 security = user。security = user 模式常用于独立文件服务器或 DC 。**

Samba 用户认证模式一共有5种，分别是 share、user、server、domain、ads。
1. share：所有人都可以访问这台 samba 服务器（不需要输入用户名和密码）。
2. user：需要输入有效的用户名和密码才能访问 samba 服务器（身份验证由 samba 服务器负责）。
3. server：与 user 相同，只是将身份验证交由指定的另一台 samba 服务器负责。
4. domain：将身份验证交由域控制器负责。
5. ads：将身份验证交由域控制器负责（比 domain 更为安全一点）。


### 创建 Samba 登陆用户
创建系统用户
```shell
$ sudo useradd -s /usr/sbin/nologin   # 禁止Linux用户登陆
$ sudo passwd usmb
```

创建 Samb 登陆用户
```shell
$ sudo smbpasswd -a usmb
```


### 创建 Samba 共享文件夹

创建 Samb 共享文件夹并设置文件夹的权限和所属用户和用户组
```shell
$ mkdir /extdisk/disk1/smbshare
$ sudo chmod -R 755 smbshare
$ sudo chown -R usmb:usmb smbshare
```


### 重启 Samba 服务
```shell
$ sudo /etc/init.d/samb restart
```

或者
```shell
$ sudo systemctl restart smbd.service
```


### 访问 Samba 共享文件夹
**1. Windows 下访问 Samba 共享文件夹**
* 在 Windows 资源管理器地址上输入  `\\+ip` （比如我的samba服务器IP地址是 `192.168.1.100` ，则输入 `\\192.168.1.100` ），登陆 Samba 服务器，
* 然后继续输入刚才设置的账号和密码就可以了。

**2. Ubuntu 16.04 下访问 Samba 共享文件夹**
* 在 Ubuntu 文件管理器上，按 `ctrl + L` 输入 `samb:// + ip` （比如我的samba服务器IP地址是 `192.168.1.100` ，则输入 `samb://192.168.1.100` ），登陆 samb 服务器，
* 然后继续输入刚才设置的账号和密码就可以了。


## 参考资料
1. [Ubuntu 下配置 Samba 服务器](https://www.jianshu.com/p/c4579605a737)
2. [Ubuntu 16.04安装配置 Samba 服务](https://blog.csdn.net/wbaction/article/details/72758673)
3. [CentOS 7 搭建 Samba 服务](https://blog.csdn.net/qq_35590198/article/details/78841036)
4. [smb.conf 官方文档](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)
