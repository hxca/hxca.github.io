---
title: Hadoop 3.12 集群部署
categories: 
- 技术经验总结
- 大数据
tags:
  - Hadoop
  - Linux
  - 大数据
abbrlink: de2f0163
date: 2020-02-18 13:27:24
---

## 前言
“闲来无聊”，搭建 Hadoop 集群来玩玩。

先来说说 Hadoop 的起源吧。在 2004 - 2005 年， Doug Cutting 受到 Google 所发表的 Google File System 和 MapReduce 论文思想启发而编写 Nutch 项目。2006 年，从 Nutch 中转移出 MapReduce 和 HDFS 组成 Hadoop 项目。后来经过多年的发展， Hadoop 已经成了一个非常丰富的生态系统。Hadoop 中最核心的两部分分别是： MapReduce 和 HDFS 。MapReduce 是并行处理框架，实现任务分解和调度。HDFS 是分布式文件系统，存储海量的数据。

在 Hadoop 官方文档中，Hadoop 有三种方式，分别是本地模式、伪分布式和完全式分布式集群。下面我搭建的是完全式分布式集群。

<!-- more -->

## Debian 系统配置
开始进入正题。我这里使用 Vmware 虚拟 4 个 Debian 系统，一个 master ，三个 solver 来搭建完全式分布式集群。hostname 分别是**master、solver1、solver2、solver3**。注意下面的JDK和hadoop安装配置操作都是使用 **hadoop 用户权限**来执行，并非 root 权限。


### 静态网络的配置
首先配置静态IP地址。其中 master 和 solver 的对应的 hostname 和 IP 地址分别为：

>master => 192.168.20.101
>solver1 => 192.168.20.102
>solver2 => 192.168.20.103
>solver3 => 192.168.20.104

分别在不同主机上配置对应的静态 IP 地址。编辑 `/etc/network/interfaces` 文件，注释自动获取 IP，并添加内容。

```
# The primary network interface
#allow-hotplug ens33
#iface ens33 inet dhcp

# static IP address
auto ens33
iface ens33 inet static
address 192.168.20.101			# 配置对应的 IP 地址
netmask 255.255.255.0			# 配置子网掩码
gateway 192.168.20.2			# 配置对应的网关
dns-nameservers 114.114.114.114		# 配置 DNS1
dns-nameservers 114.114.115.115		# 配置 DNS2
```

### hosts文件配置
修改 `/etc/hosts` 文件，为每个主机通过 hostname 访问。添加内容
```
# Hadoop hosts
192.168.20.101  master
192.168.20.102  solver1
192.168.20.103  solver2
192.168.20.104  solver3
```

### 安装 `openssh-server` 和 `vim`
```shell
# apt-get install openssh-server vim
```


### 创建用户和用户组
创建 Hadoop 专用的用户 `hadoop` 。在每台主机上执行命令：
```shell
# useradd -m -s /bin/bash hadoop
```

### 生成ssh密钥
分别在不同的主机上执行命令
```shell
# master
$ ssh-keygen -t rsa -C "master"

# solver1
$ ssh-keygen -t rsa -C "solver1"

# solver2
$ ssh-keygen -t rsa -C "solver2"

# solver3
$ ssh-keygen -t rsa -C "solver3"
```

### 免密码登录
在每台主机上执行命令
```shell
$ ssh-copy-id -i ~/.ssh/id_rsa.pub master
$ ssh-copy-id -i ~/.ssh/id_rsa.pub solver1
$ ssh-copy-id -i ~/.ssh/id_rsa.pub solver2
$ ssh-copy-id -i ~/.ssh/id_rsa.pub solver3
```

## JDK 安装与配置

### 手动安装 JDK
解压 JDK 包安装包到 `/usr/lib/jvm/` ，然后创建 `jdk` 软链接指向 `jdk1.8.0_202` 目录
```shell
$ sudo ln -sf /usr/lib/jvm/jdk1.8.0_202 /usr/lib/jvm/jdk
```

### JDK 环境变量的配置
在 `/etc/profile.d` 目录下新建 `jdk.sh` 文件
```shell
$ sudo vi /etc/profile.d/jdk.sh
```

写入内容
```
# JDK environment settings
export JAVA_HOME=/usr/lib/jvm/jdk
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATh=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

刷新 `/etc/profile` 环境变量
```shell
$ source /etc/profile
```

验证 JAVA 是否正确安装
```shell
$ java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)
```

***把 jdk 安装包和 jdk.sh 分别 scp 到每台主机上，重复上面的操作。***

## Hadoop 安装与配置

### Hadoop 安装

解压 hadoop 安装包到 `/opt` ，并创建 `hadoop` 软链接指向 `hadoop-3.1.2` 目录
```shell
$ sudo ln -sf /opt/hadoop-3.1.2 /opt/hadoop
```

在 `hadoop` 下分别创建 `logs` 、 `hdfs/name` 、 `hdfs/data` 文件夹
```shell
$ sudo mkdir /opt/hadoop/logs
$ sudo mkdir -p /opt/hadoop/hdfs/name
$ sudo mkdir -p /opt/hadoop/hdfs/data
```

修改 `hadoop-3.1.2` 目录的拥有者
```shell
$ sudo chown -R hadoop:hadoop /opt/hadoop-3.1.2
```

### hadoop 环境变量的配置

在 `/etc/profile.d` 目录下新建文件`hadoop.sh`
```shell
$ sudo vi /etc/profile.d/hadoop.sh
```

添加如下内容
```
# Hadoop environment settings
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

刷新 `/etc/profile` 变量
```shell
$ source /etc/profile
```

### Hadoop 文件配置
Hadoop 的配置文件都在 `etc/hadoop/` 文件夹下，分别编辑  `hadoop-env.sh` ，`workers`  ， `core-site.xml ` ， `hdfs-site.xml`， `mapred-site.xml` ， `yarn-site.xml`

`hadoop-env.sh`
由于远程调用 `${java_home}` 时，通常出现找不到jdk变量。所以需要手动配置hadoop的jdk环境变量。编辑 `hadoop-env.sh` 文件，并输入内容。
```
export JAVA_HOME=/usr/lib/jvm/jdk
```

`workers`
添加所有 solver 机器的hostname到 `workers`
```
solver1
solver2
solver3
```

`core-site.xml`
```xml
<configuration>

  <!-- hdfs的位置 -->
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
  </property>

  <!-- hadoop运行时产生的缓冲文件存储位置 -->
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop/tmp</value>
  </property>

</configuration>
```


`hdfs-site.xml`
```xml
<configuration>

  <!-- hdfs 数据备份数量 -->
  <property>
   <name>dfs.replication</name>
    <value>1</value>
  </property>

  <!-- hdfs namenode上存储hdfs名字空间元数据 -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/opt/hadoop/hdfs/name</value>
  </property>

  <!-- hdfs datanode上数据块的物理存储位置 -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/opt/hadoop/hdfs/data</value>
  </property>

</configuration>
```

`mapred-site.xml`
```xml
<configuration>

  <!--  mapreduce运行的平台 默认local本地模式 -->
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>

  <!--  mapreduce web UI address -->
  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>master:19888</value>
  </property>

</configuration>
```

`yarn-site.xml`
```xml
<configuration>
  <!-- Site specific YARN configuration properties -->

  <!--  yarn 的 hostname -->
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
  </property>

  <!--  yarn Web UI address -->
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>${yarn.resourcemanager.hostname}:8088</value>
  </property>

  <!--  reducer 获取数据的方式 -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>

</configuration>
```




***把 `/opt/hadoop-3.1.2` 和 `hadoop.sh` 打包 scp 到每台电脑上，然后配置 Hadoop 环境变量和创建 `/opt/hadoop` 指向 `/opt/hadoop-3.1.2`***

##  Hadoop 的验证

### 启动 Hadoop 集群

在启动 Hadoop 之前，首先格式化 hdfs
```shell
$ hdfs namenode -format
```

启动 jobhistoryserver
```shell
$ mr-jobhistory-daemon.sh start historyserver
```
与之相应的关闭
```shell
$ mr-jobhistory-daemon.sh stop historyserver
```

启动 yarn
```shell
$ start-yarn.sh
```

与之相对应的关闭
```shell
$ stop-yarn.sh
```

启动 hdfs
```shell
$ start-dfs.sh
```

与之相对应的关闭
```shell
$ stop-dfs.sh
```

**或者一键启动与关闭**
```shell
start-all.sh
stop-all.sh
```

**最后验证 Hadoop 是否成功启动**
```shell
$ jps
13074 SecondaryNameNode
14485 Jps
10441 JobHistoryServer
12876 NameNode
13341 ResourceManager
```

如果输入 `jps` 后，有 `SecondaryNameNode` ， `Jps` ， `JobHistoryServer` ， `NameNode` ， `ResourceManager` 相关信息输出，那么说明Hadoop启动成功。


### Hadoop Web UI 访问

| Daemon 						| Web Interface 				| Notes 						|
|  ---------------------------- | ----------------------------- | ----------------------------- |
| NameNode 						| https://192.168.20.101:9870 	| Default HTTP port is 9870. 	|
| Resourcemanager 				| http://192.168.20.101:8088 	| Default HTTP port is 8088. 	|
| MapReduce JobHistory Server 	| http://192.168.20.101:19888 	| Default HTTP port is 19888. 	|

